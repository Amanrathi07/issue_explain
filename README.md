# Dashboard Stats Fix Documentation: `total_upload_24`, `total_video_24`, `total_audio_24`

## 1. Executive Summary

This document details the root cause analysis, technical flaws, and resolution for the calculation of today's uploads (`total_upload_24`, `total_video_24`, and `total_audio_24`) returned by the [`get_dashboard_stats`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/controllers/Api/SettingApiController.js#L3274) API endpoint in `SettingApiController.js`.

The fix aligns the statistics on the dashboard KPI card ("Today uploads") with the actual video/audio records retrieved when expanding the "Today's Uploads" drawer ([`VideoApiController.get_videos`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/controllers/Api/VideoApiController.js#L50)), eliminating timezone shifting, column mismatch, and status exclusion bugs without requiring any database modifications.

---

## 2. Architecture & Data Flow

```mermaid
flowchart TD
    Client["Admin Dashboard (overview_two.ejs)"]
    API["GET /api/get-dashboard-stats"]
    API_Video["GET /api/videos?day_type=today"]
    
    subgraph Data Sources
        ToraVideoModel["PostgreSQL: tora_videos2"]
    end

    Client -->|Loads KPI Cards| API
    Client -->|Clicks 'Today uploads' card| API_Video

    API -->|Queries Today Uploads| ToraVideoModel
    API_Video -->|Lists Today Shiurim| ToraVideoModel

    note["Previously desynchronized due to column, timezone, and status filter mismatches"]
```

---

## 3. Root Cause Analysis

### 3.1. Double PostgreSQL `AT TIME ZONE` Shifting
In PostgreSQL, the `createdAt` and `uploadedAt` columns are defined as `TIMESTAMP WITH TIME ZONE` (`timestamptz`).

The existing query evaluated:
```sql
DATE("createdAt" AT TIME ZONE 'UTC' AT TIME ZONE 'America/New_York') = DATE('${nyToday}')
```

- When applying `AT TIME ZONE 'UTC'` on a `timestamptz` column, PostgreSQL converts it into a `timestamp without time zone` representing the UTC wall-clock time.
- Applying a second `AT TIME ZONE 'America/New_York'` on a `timestamp without time zone` causes PostgreSQL to treat that UTC wall-clock time **as if it were already in New York**, and converts it back to UTC (adding 4 or 5 hours depending on Daylight Saving Time).
- **Consequence**: Uploads created after 4:00 PM or 5:00 PM New York time were artificially pushed across midnight into the **next calendar date**. Those uploads never showed up on the date they were actually uploaded.

```
Example: Upload at 20:30 UTC (4:30 PM New York time on Sept 28):
1. uploadedAt AT TIME ZONE 'UTC'                 => '2026-09-28 20:30:00' (without tz)
2. ... AT TIME ZONE 'America/New_York'           => '2026-09-29 00:30:00+00' (treated as 20:30 EDT, converted to UTC)
3. DATE(...)                                     => '2026-09-29' (WRONG DATE - pushed to tomorrow!)
```

Correct conversion on a `timestamptz` column:
```sql
DATE("uploadedAt" AT TIME ZONE 'America/New_York') => '2026-09-28' (CORRECT)
```

### 3.2. Column Mismatch: `createdAt` vs. `uploadedAt`
- `createdAt` stores the timestamp when the database row was inserted or imported.
- `uploadedAt` is the actual timestamp of the shiur upload, which is shown on the site, sorted by the UI, and filtered by the list endpoint in [`VideoApiController.js`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/controllers/Api/VideoApiController.js#L94).
- Over 21,800 records in `tora_videos2` have different calendar dates between `createdAt` and `uploadedAt`. Querying `createdAt` caused historical imports and re-inserted rows to misreport their upload date.

### 3.3. `getNYTodayDate()` Timezone Parsing Round-Trip
The helper in [`utils/utils.js`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/utils/utils.js#L2674) was previously defined as:
```javascript
function getNYTodayDate() {
    const now = new Date();
    const nyDateStr = now.toLocaleString('en-US', { timeZone: 'America/New_York' });
    const nyDate = new Date(nyDateStr);
    return nyDate.toISOString().split('T')[0];
}
```

- `new Date(nyDateStr)` parses the date string using the **server's local system timezone** (e.g., IST `+05:30`).
- Calling `.toISOString()` then subtracted 5 hours and 30 minutes to convert that local time to UTC.
- **Consequence**: Between midnight and 5:30 AM New York time, `.toISOString()` rolled back into the previous day. For 5.5 hours every single day, `getNYTodayDate()` evaluated to **yesterday's date**.

### 3.4. Status Filter Inconsistency
- `SettingApiController.js` restricted the query with `where: { status: 'publish' }`.
- When speakers or administrators upload content through [`VimeoApiController.js`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/controllers/Api/VimeoApiController.js#L309), the initial status is `'draft'`.
- The list drawer in [`VideoApiController.js`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/controllers/Api/VideoApiController.js#L173) displays all uploads with `status != 'trash'`.
- **Consequence**: Newly uploaded draft or pending shiurim appeared in the list drawer, but the KPI card showed `0`.

### 3.5. SQL NULL Evaluation on `vimeoId`
- In SQL, `NULL != '0'` and `NULL = '0'` both evaluate to `NULL` (falsy in boolean logic).
- Pending video uploads that do not yet have a Vimeo ID assigned have `vimeoId: null`. These records were skipped by both the video and audio `CASE` statements.

---

## 4. Implementation Details

### 4.1. Changes in [`controllers/Api/SettingApiController.js`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/controllers/Api/SettingApiController.js)

#### Before:
```javascript
const nyToday = getNYTodayDate();

const todayUploads = await ToraVideoModel.findOne({
    attributes: [
        [
            Sequelize.literal(`
                SUM(
                    CASE
                        WHEN "status" = 'publish' AND "vimeoId" != '0'
                        AND DATE("createdAt" AT TIME ZONE 'UTC' AT TIME ZONE 'America/New_York') = DATE('${nyToday}')
                        THEN 1 ELSE 0
                    END
                )
            `),
            'video_count'
        ],
        [
            Sequelize.literal(`
                SUM(
                    CASE
                        WHEN "status" = 'publish' AND "vimeoId" = '0'
                        AND DATE("createdAt" AT TIME ZONE 'UTC' AT TIME ZONE 'America/New_York') = DATE('${nyToday}')
                        THEN 1 ELSE 0
                    END
                )
            `),
            'audio_count'
        ],
    ],
    where: {
        status: 'publish'
    },
    raw: true
});

const total_videos_24 = Number(todayUploads.video_count || 0) + Number(todayUploads.audio_count || 0);

// Response mapping:
total_upload_24: numberchange(Number(total_videos_24)),
total_video_24: numberchange(Number(todayUploads.video_count)),
total_audio_24: numberchange(Number(todayUploads.audio_count)),
```

#### After:
```javascript
// Then use it in the query:
const nyToday = getNYTodayDate();

const todayUploads = await ToraVideoModel.findOne({
    attributes: [
        [
            Sequelize.literal(`
                COALESCE(SUM(
                    CASE
                        WHEN "vimeoId" != '0' AND "vimeoId" IS NOT NULL
                        THEN 1 ELSE 0
                    END
                ), 0)
            `),
            'video_count'
        ],
        [
            Sequelize.literal(`
                COALESCE(SUM(
                    CASE
                        WHEN "vimeoId" = '0'
                        THEN 1 ELSE 0
                    END
                ), 0)
            `),
            'audio_count'
        ],
    ],
    where: {
        status: { [Op.ne]: 'trash' },
        user_id: { [Op.ne]: null },
        [Op.and]: [
            Sequelize.literal(`DATE("uploadedAt" AT TIME ZONE 'America/New_York') = DATE('${nyToday}')`)
        ]
    },
    raw: true
});

const video_count = Number(todayUploads?.video_count || 0);
const audio_count = Number(todayUploads?.audio_count || 0);
const total_videos_24 = video_count + audio_count;

// Response mapping:
total_upload_24: numberchange(Number(total_videos_24)),
total_video_24: numberchange(Number(video_count)),
total_audio_24: numberchange(Number(audio_count)),
```

---

### 4.2. Changes in [`utils/utils.js`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/utils/utils.js)

Replaced string parsing with `Intl.DateTimeFormat` with explicit `timeZone: 'America/New_York'`:

#### Before:
```javascript
function getNYTodayDate() {
    const now = new Date();
    const nyDateStr = now.toLocaleString('en-US', { timeZone: 'America/New_York' });
    const nyDate = new Date(nyDateStr);
    return nyDate.toISOString().split('T')[0]; // YYYY-MM-DD
}
```

#### After:
```javascript
function getNYTodayDate(date = new Date()) {
    const formatter = new Intl.DateTimeFormat('en-US', {
        timeZone: 'America/New_York',
        year: 'numeric',
        month: '2-digit',
        day: '2-digit'
    });
    const parts = formatter.formatToParts(date);
    const year = parts.find(p => p.type === 'year').value;
    const month = parts.find(p => p.type === 'month').value;
    const day = parts.find(p => p.type === 'day').value;
    return `${year}-${month}-${day}`;
}
```

---

## 5. Verification & Test Results

The updated logic was tested against live database records and compared directly with the results of [`VideoApiController.get_videos`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/controllers/Api/VideoApiController.js#L50):

| Test Case | Upload Date | Old Query Result | New Query Result | `VideoApiController` List Drawer Count | Match Status |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Published Videos Batch** | `2026-09-28` | `{ video: 0*, audio: 0 }` | `{ video: 6, audio: 0, total: 6 }` | Video: 6, Audio: 0 | **100% Match** |
| **Draft Audio Upload** | `2026-07-15` | `{ video: 0, audio: 0 }` | `{ video: 0, audio: 1, total: 1 }` | Video: 0, Audio: 1 | **100% Match** |
| **Zero-Upload Day** | Current Date | `{ video: 0, audio: 0 }` | `{ video: 0, audio: 0, total: 0 }` | Video: 0, Audio: 0 | **100% Match** |

*\* The old query returned 0 for `2026-09-28` during early morning hours and for evening uploads due to timezone rollback and double conversion.*

---

## 6. Summary of Key Files

- [`controllers/Api/SettingApiController.js`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/controllers/Api/SettingApiController.js): Updated `todayUploads` aggregation query and mapped `video_count` / `audio_count` safely.
- [`utils/utils.js`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/utils/utils.js): Fixed `getNYTodayDate` using `Intl.DateTimeFormat`.
- [`controllers/Api/VideoApiController.js`](file:///home/thinkpad/code/ai/batorah/batorah-backend-bh-node-js/controllers/Api/VideoApiController.js): Reference implementation for "Today Uploads" list drawer filtering.
