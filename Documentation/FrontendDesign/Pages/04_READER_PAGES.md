# Reader Pages

## 1. Document Information

| Attribute | Value |
|-----------|-------|
| Version | 2.0 |
| Last Updated | 2026-01-04 |
| Status | Approved |
| Owner | Frontend Engineering Team |
| Review Cycle | Quarterly |
| Backend Reference | [05_READINGS_COMPONENT.md](../../../../Backend/N9/Documentation/BackendDesign/Components/05_READINGS_COMPONENT.md), [01_STORIES_COMPONENT.md](../../../../Backend/N9/Documentation/BackendDesign/Components/01_STORIES_COMPONENT.md) |

---

## 2. Overview

### 2.1 Purpose
This document specifies the **reader pages** for the N9 platform, including the immersive chapter reader, reading settings, library management, and offline reading. Aligned with backend Readings and Stories components.

### 2.2 User Flows

```
┌─────────────────────────────────────────────────────────────────┐
│                      READER FLOWS                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Reading Flow:                                                   │
│  Story Detail → Chapter → Read → Next Chapter → Continue         │
│                                                                  │
│  Settings Flow:                                                  │
│  Reader → Settings Panel → Adjust → Auto-save → Continue         │
│                                                                  │
│  Navigation Flow:                                                │
│  Reader → Table of Contents → Select Chapter → Jump              │
│                                                                  │
│  Bookmark Flow:                                                  │
│  Reader → Add Bookmark → Library → View Bookmarks → Jump         │
│                                                                  │
│  Highlight Flow:                                                 │
│  Select Text → Highlight → Add Note → Save                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Related Backend APIs

#### Chapter Content Endpoints
| Endpoint | Method | Purpose |
|----------|--------|----------|
| `/api/v1/chapters/{id}` | GET | Chapter content |
| `/api/v1/chapters/{id}/unlock` | POST | Unlock premium chapter |
| `/api/v1/stories/{storyId}/chapters/batch` | GET | Batch download chapters |

#### Reading Progress Endpoints
| Endpoint | Method | Purpose |
|----------|--------|----------|
| `/api/v1/reading-progress` | GET | Get all progress |
| `/api/v1/reading-progress` | POST | Save reading position |
| `/api/v1/reading-progress/sync` | POST | Sync progress across devices |
| `/api/v1/reading-progress/story/{storyId}` | GET | Progress for story |
| `/api/v1/reading-progress/history` | GET | Reading history |

#### Bookmarks & Highlights Endpoints
| Endpoint | Method | Purpose |
|----------|--------|----------|
| `/api/v1/bookmarks` | GET | List bookmarks |
| `/api/v1/bookmarks` | POST | Create bookmark |
| `/api/v1/bookmarks/{id}` | DELETE | Delete bookmark |
| `/api/v1/highlights` | GET | List highlights |
| `/api/v1/highlights` | POST | Create highlight |
| `/api/v1/highlights/{id}` | PUT | Update highlight note |
| `/api/v1/highlights/{id}` | DELETE | Delete highlight |

#### Library Endpoints
| Endpoint | Method | Purpose |
|----------|--------|----------|
| `/api/v1/library` | GET | User's library |
| `/api/v1/library/collections` | GET | User's collections |
| `/api/v1/library/collections` | POST | Create collection |
| `/api/v1/library/collections/{id}` | PUT | Update collection |

---

## 3. Routes Configuration

```typescript
// Reader routes
const readerRoutes = [
  { 
    path: '/read/:storySlug/:chapterNumber', 
    element: <ReaderPage />,
    layout: null, // No standard layout - immersive mode
  },
  { path: '/library', element: <LibraryPage />, auth: true },
  { path: '/library/history', element: <ReadingHistoryPage />, auth: true },
  { path: '/library/bookmarks', element: <BookmarksPage />, auth: true },
  { path: '/library/collections', element: <CollectionsPage />, auth: true },
  { path: '/library/collections/:id', element: <CollectionDetailPage />, auth: true },
];
```

---

## 4. Chapter Reader Page

### 4.1 Specification

| Attribute | Value |
|-----------|-------|
| Route | `/read/:storySlug/:chapterNumber` |
| Auth Required | No (free) / Yes (premium, features) |
| Mobile Support | Full (touch optimized) |
| SEO | Minimal (content protected) |

### 4.2 User Story
> As a reader, I want an immersive, distraction-free reading experience with customizable settings so I can enjoy the story comfortably.

### 4.3 Layout - Reading Mode

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  [← Back]        Chapter 45: The Revelation        [⚙️] [☰]    │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                                                                  │
│                       Chapter 45                                 │
│                    The Revelation                                │
│                                                                  │
│                                                                  │
│      The sun had barely risen over the mountains when           │
│  Elena opened her eyes. Something felt different today—         │
│  the air itself seemed to hum with an ancient energy            │
│  she had never noticed before.                                  │
│                                                                  │
│      She sat up in her bed, her heart racing. The               │
│  dreams had been vivid again, filled with images of             │
│  dragons soaring through clouds of gold and crimson.            │
│  But this time, she had heard a voice calling her               │
│  name, beckoning her toward something she couldn't              │
│  quite understand.                                              │
│                                                                  │
│      "Elena," the voice had whispered, soft as the              │
│  wind through the ancient trees. "The time has come."           │
│                                                                  │
│      She rose from her bed and walked to the window,            │
│  pulling back the heavy curtain. The view that greeted          │
│  her was breathtaking—rolling hills covered in morning          │
│  mist, with the distant peaks of the Dragon Mountains           │
│  gleaming in the early light.                                   │
│                                                                  │
│      Today would change everything. She could feel it           │
│  in her bones, in the way the magic stirred within              │
│  her chest. After eighteen years of waiting, the                │
│  answers she sought were finally within reach.                  │
│                                                                  │
│      ...                                                        │
│                                                                  │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ████████████████░░░░░░░░░░░░░░░░░░░░░  35%                     │
│                                                                  │
│  [← Ch 44]              45/125              [Ch 46 →]           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 Layout - Mobile

```
┌─────────────────────────────────┐
│ ← 45: The Revelation      ⚙️ ☰ │
├─────────────────────────────────┤
│                                 │
│      Chapter 45                 │
│   The Revelation                │
│                                 │
│                                 │
│   The sun had barely risen      │
│   over the mountains when       │
│   Elena opened her eyes.        │
│   Something felt different      │
│   today—the air itself seemed   │
│   to hum with an ancient        │
│   energy she had never          │
│   noticed before.               │
│                                 │
│   She sat up in her bed, her    │
│   heart racing. The dreams      │
│   had been vivid again,         │
│   filled with images of         │
│   dragons soaring through       │
│   clouds of gold and crimson.   │
│                                 │
│   ...                           │
│                                 │
│                                 │
├─────────────────────────────────┤
│ ████████░░░░░░░░░░░░░░░░  35%  │
│                                 │
│   [←]        45/125        [→]  │
│                                 │
└─────────────────────────────────┘
```

### 4.5 Component Hierarchy

```tsx
<ReaderLayout theme={settings.theme}>
  <ReaderHeader>
    <BackButton />
    <ChapterTitle>{chapter.title}</ChapterTitle>
    <HeaderActions>
      <SettingsButton onClick={openSettings} />
      <TOCButton onClick={openTOC} />
    </HeaderActions>
  </ReaderHeader>
  
  <ReaderContent
    ref={contentRef}
    onScroll={handleScroll}
    style={{
      fontSize: settings.fontSize,
      fontFamily: settings.fontFamily,
      lineHeight: settings.lineHeight,
    }}
  >
    <ChapterHeader>
      <h1>{chapter.title}</h1>
      {chapter.subtitle && <p>{chapter.subtitle}</p>}
    </ChapterHeader>
    
    <ChapterBody 
      content={chapter.content}
      onTextSelect={handleTextSelect}
    />
    
    <ChapterFooter>
      <ChapterNotes notes={chapter.authorNotes} />
      <ChapterReactions />
    </ChapterFooter>
  </ReaderContent>
  
  <ReaderFooter>
    <ProgressBar progress={readProgress} />
    <NavigationControls>
      <PrevChapterButton chapter={prevChapter} />
      <ChapterIndicator current={chapter.number} total={totalChapters} />
      <NextChapterButton chapter={nextChapter} />
    </NavigationControls>
  </ReaderFooter>
  
  {/* Overlays */}
  <SettingsPanel isOpen={settingsOpen} onClose={closeSettings} />
  <TOCPanel isOpen={tocOpen} onClose={closeTOC} />
  <TextSelectionMenu selection={selection} />
</ReaderLayout>
```

### 4.6 Reading Settings

```
┌─────────────────────────────────────────────────────────────────┐
│                    Reading Settings                        [✕]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  APPEARANCE                                                      │
│                                                                  │
│  Theme                                                           │
│  ┌────────┐ ┌────────┐ ┌────────┐                               │
│  │  ☀️    │ │  🌙    │ │  📜    │                               │
│  │ Light  │ │  Dark  │ │ Sepia  │                               │
│  │  [●]   │ │  [ ]   │ │  [ ]   │                               │
│  └────────┘ └────────┘ └────────┘                               │
│                                                                  │
│  ────────────────────────────────────────────────────────────   │
│                                                                  │
│  TYPOGRAPHY                                                      │
│                                                                  │
│  Font Size                                                       │
│  [A-] ━━━━━━━━●━━━━━━━━━━ [A+]          18px                    │
│                                                                  │
│  Font Family                                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ System Default                                        ▼ │    │
│  └─────────────────────────────────────────────────────────┘    │
│  Options: System Default, Serif, Sans-serif, OpenDyslexic       │
│                                                                  │
│  Line Height                                                     │
│  [≡] ━━━━━━━━━━●━━━━━━━━━ [≡]          1.8                     │
│                                                                  │
│  Text Alignment                                                  │
│  ┌────┐ ┌────┐ ┌────┐                                          │
│  │ ≡← │ │ ≡≡ │ │ →≡ │                                          │
│  │Left│ │Just│ │Rght│                                          │
│  │[●] │ │[ ] │ │[ ] │                                          │
│  └────┘ └────┘ └────┘                                          │
│                                                                  │
│  ────────────────────────────────────────────────────────────   │
│                                                                  │
│  DISPLAY                                                         │
│                                                                  │
│  Page Width                                                      │
│  [◇] ━━━━━━━━━●━━━━━━━━━━ [◈]          720px                   │
│                                                                  │
│  Page Margins                                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Medium                                                ▼ │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ────────────────────────────────────────────────────────────   │
│                                                                  │
│  READING                                                         │
│                                                                  │
│  ☑ Show progress bar                                            │
│  ☑ Auto-save reading position                                   │
│  ☐ Auto-scroll (beta)                                           │
│  ☑ Tap zones for navigation                                     │
│                                                                  │
│  ────────────────────────────────────────────────────────────   │
│                                                                  │
│  [Reset to Defaults]                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.7 Reading Settings State

```typescript
interface ReaderSettings {
  // Appearance
  theme: 'light' | 'dark' | 'sepia';
  
  // Typography
  fontSize: number; // 12-32px
  fontFamily: 'system' | 'serif' | 'sans-serif' | 'opendyslexic';
  lineHeight: number; // 1.2-2.5
  textAlign: 'left' | 'justify' | 'right';
  
  // Display
  pageWidth: number; // 480-1200px
  margins: 'none' | 'small' | 'medium' | 'large';
  
  // Reading
  showProgressBar: boolean;
  autoSavePosition: boolean;
  autoScroll: boolean;
  autoScrollSpeed: number;
  tapZones: boolean;
}

// Settings store (persisted)
const useReaderSettings = create(
  persist<ReaderSettings>(
    (set) => ({
      theme: 'light',
      fontSize: 18,
      fontFamily: 'system',
      lineHeight: 1.8,
      textAlign: 'left',
      pageWidth: 720,
      margins: 'medium',
      showProgressBar: true,
      autoSavePosition: true,
      autoScroll: false,
      autoScrollSpeed: 50,
      tapZones: true,
    }),
    { name: 'reader-settings' }
  )
);
```

### 4.8 Table of Contents Panel

```
┌─────────────────────────────────────────────────────────────────┐
│                   Table of Contents                        [✕]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 🔍 Search chapters...                                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Volume 1: The Awakening                                         │
│  ├─ Ch 1: The Beginning                           ✓ Read        │
│  ├─ Ch 2: A Strange Dream                         ✓ Read        │
│  ├─ Ch 3: The Academy                             ✓ Read        │
│  ├─ ...                                                         │
│  └─ Ch 25: The First Test                         ✓ Read        │
│                                                                  │
│  Volume 2: The Journey                                           │
│  ├─ Ch 26: Setting Out                            ✓ Read        │
│  ├─ Ch 27: The Mountain Pass                      ✓ Read        │
│  ├─ ...                                                         │
│  ├─ Ch 44: The Ancient Temple                     ✓ Read        │
│  ├─ Ch 45: The Revelation                         ● Current     │
│  ├─ Ch 46: Aftermath                              ○ Unread      │
│  └─ Ch 50: The Decision                           ○ Unread      │
│                                                                  │
│  Volume 3: The Battle                                            │
│  ├─ Ch 51: War Begins                             🔒 5 🪙       │
│  ├─ ...                                                         │
│  └─ Ch 75: Victory's Price                        🔒 5 🪙       │
│                                                                  │
│  ...                                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.9 Text Selection Menu

```
         ┌─────────────────────────────────────────┐
         │  📝 Note │ 🖍️ Highlight │ 📋 Copy │ 💬  │
         └─────────────────────────────────────────┘
                            ▲
                    (selected text)
```

### 4.10 Highlight Colors

```tsx
<HighlightColorPicker>
  <ColorOption color="yellow" selected />
  <ColorOption color="green" />
  <ColorOption color="blue" />
  <ColorOption color="pink" />
  <ColorOption color="purple" />
</HighlightColorPicker>
```

### 4.11 Navigation Interactions

| Action | Desktop | Mobile |
|--------|---------|--------|
| Previous chapter | Left arrow key / Click button | Swipe right / Tap left zone |
| Next chapter | Right arrow key / Click button | Swipe left / Tap right zone |
| Scroll | Mouse wheel / Page Up/Down | Touch scroll |
| Open TOC | Press 'T' / Click button | Tap button |
| Open settings | Press 'S' / Click button | Tap button |
| Toggle UI | Press 'H' / Click content | Tap content |
| Add bookmark | Press 'B' | Double-tap |
| Exit reader | Press 'Escape' / Click back | Swipe down / Tap back |

### 4.12 Reading Progress Sync

```typescript
// Auto-save reading position
function useReadingProgress(chapterId: string) {
  const [progress, setProgress] = useState(0);
  const debouncedProgress = useDebounce(progress, 2000);
  
  // Save progress when it changes
  useEffect(() => {
    if (debouncedProgress > 0) {
      api.readingProgress.save({
        chapterId,
        progress: debouncedProgress,
        timestamp: Date.now(),
      });
    }
  }, [debouncedProgress, chapterId]);
  
  // Handle scroll
  const handleScroll = useCallback((e: UIEvent) => {
    const el = e.target as HTMLElement;
    const scrollProgress = el.scrollTop / (el.scrollHeight - el.clientHeight);
    setProgress(Math.round(scrollProgress * 100));
  }, []);
  
  return { progress, handleScroll };
}
```

---

## 5. Premium Chapter Unlock

### 5.1 Locked Chapter View

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  [← Back]        Chapter 51: War Begins           [⚙️] [☰]      │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                                                                  │
│                          🔒                                      │
│                                                                  │
│                  This chapter is locked                          │
│                                                                  │
│                      5 🪙 coins                                  │
│                                                                  │
│       ┌─────────────────────────────────────┐                   │
│       │         Unlock This Chapter         │                   │
│       └─────────────────────────────────────┘                   │
│                                                                  │
│       ┌─────────────────────────────────────┐                   │
│       │    Unlock All Remaining (375 🪙)    │                   │
│       └─────────────────────────────────────┘                   │
│                                                                  │
│                                                                  │
│              Your balance: 250 🪙                                │
│              [Top Up Coins]                                      │
│                                                                  │
│                                                                  │
│  ─────────────────────────────────────────────────────────────  │
│                                                                  │
│  Preview (first 500 words):                                      │
│                                                                  │
│      The drums of war echoed through the valley as              │
│  General Marcus surveyed the battlefield below. His             │
│  army, twenty thousand strong, stood ready to face              │
│  the enemy forces that had gathered on the opposite             │
│  hillside...                                                    │
│                                                                  │
│  [Unlock to continue reading →]                                 │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [← Ch 50]              51/125              [Ch 52 🔒]          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Unlock Confirmation Modal

```
┌─────────────────────────────────────────────────────────────────┐
│                   Unlock Chapter                           [✕]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Chapter 51: War Begins                                          │
│                                                                  │
│  Cost: 5 🪙                                                      │
│  Your balance: 250 🪙                                            │
│                                                                  │
│  ────────────────────────────────────────────────────────────   │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Unlock (5 🪙)                                  ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│                      [Cancel]                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. Chapter Comments

### 6.1 Comments Panel

```
┌─────────────────────────────────────────────────────────────────┐
│                   Chapter Comments (234)                   [✕]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ Write a comment...                                          ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  Sort: [Top ▼]   ☑ Hide spoilers                               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 👤 ReaderName                              2 hours ago      ││
│  │                                                              ││
│  │ "This chapter was incredible! The plot twist at the end     ││
│  │ completely caught me off guard. Can't wait for the next one!"││
│  │                                                              ││
│  │ 👍 45  💬 3 replies     [Reply]                             ││
│  │                                                              ││
│  │   └─ 👤 AnotherReader                      1 hour ago       ││
│  │      "Right?! I had to read it twice to make sure I         ││
│  │      understood what happened!"                              ││
│  │      👍 12     [Reply]                                      ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 👤 User123              ⚠️ Spoiler        3 hours ago       ││
│  │                                                              ││
│  │ [Click to reveal spoiler]                                   ││
│  │                                                              ││
│  │ 👍 23  💬 5 replies     [Reply]                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  [Load More Comments]                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Library Page

### 7.1 Specification

| Attribute | Value |
|-----------|-------|
| Route | `/library` |
| Auth Required | Yes |
| Mobile Support | Full |
| SEO | Minimal |

### 7.2 Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  [Logo] [Home] [Library] [Browse]         🔍   🔔(3)  [Avatar▼]│
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  MY LIBRARY                                                      │
│                                                                  │
│  [Reading] [Plan to Read] [Completed] [On Hold] [Dropped] [All] │
│                                                                  │
│  🔍 Search library...                    View: [Grid] [List]    │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CURRENTLY READING (12)                                          │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ┌─────┐ The Dragon's Legacy              Progress: 35%     ││
│  │ │Cover│ by AuthorName                    Ch 45/125         ││
│  │ │     │ Last read: 2 hours ago           ████████░░░░░░░░  ││
│  │ └─────┘                                                     ││
│  │         📖 New chapter available!        [Continue Reading] ││
│  │ ─────────────────────────────────────────────────────────── ││
│  │ ┌─────┐ The Secret Garden                Progress: 72%     ││
│  │ │Cover│ by Jane Author                   Ch 36/50          ││
│  │ │     │ Last read: Yesterday             ██████████████░░░ ││
│  │ └─────┘                                  [Continue Reading] ││
│  │ ─────────────────────────────────────────────────────────── ││
│  │ ┌─────┐ City of Stars                    Progress: 15%     ││
│  │ │Cover│ by StarWriter                    Ch 8/52           ││
│  │ │     │ Last read: 3 days ago            ███░░░░░░░░░░░░░░ ││
│  │ └─────┘                                  [Continue Reading] ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  [Show All →]                                                    │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  COLLECTIONS                                           [Manage]  │
│                                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │ ❤️        │ │ 🌟        │ │ 📚        │ │ ➕        │           │
│  │ Favorites│ │ Fantasy  │ │ Weekend  │ │ Create   │           │
│  │   23     │ │ Gems 15  │ │ Reads 8  │ │ New      │           │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘           │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  READING STATISTICS                                              │
│                                                                  │
│  This Week          This Month         All Time                  │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
│  │ 📖 15 chapters│  │ 📖 62 chapters│  │ 📖 1,234 chaps│        │
│  │ ⏱️ 8.5 hours  │  │ ⏱️ 34 hours   │  │ ⏱️ 520 hours  │        │
│  └───────────────┘  └───────────────┘  └───────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 Library Tab States

| Tab | Description | Count Source |
|-----|-------------|--------------|
| Reading | Currently reading | In-progress stories |
| Plan to Read | Saved for later | Queued stories |
| Completed | Finished stories | Completed readings |
| On Hold | Paused reading | Paused stories |
| Dropped | Abandoned | Dropped stories |
| All | All library items | Total count |

### 7.4 Component Hierarchy

```tsx
<PageLayout>
  <PageHeader>
    <h1>My Library</h1>
  </PageHeader>
  
  <TabNavigation 
    tabs={['Reading', 'Plan to Read', 'Completed', 'On Hold', 'Dropped', 'All']}
    counts={tabCounts}
  />
  
  <LibraryFilters>
    <SearchInput placeholder="Search library..." />
    <ViewToggle options={['grid', 'list']} />
  </LibraryFilters>
  
  <TabContent>
    <LibrarySection title="Currently Reading" count={readingCount}>
      {readingStories.map(story => (
        <LibraryStoryCard
          key={story.id}
          story={story}
          progress={story.progress}
          hasUpdate={story.hasNewChapter}
        />
      ))}
    </LibrarySection>
  </TabContent>
  
  <CollectionsSection>
    <SectionHeader title="Collections" action="Manage" />
    <CollectionGrid collections={collections} />
  </CollectionsSection>
  
  <ReadingStats stats={readingStats} />
</PageLayout>
```

### 7.5 Library Item Actions

| Action | Trigger | Result |
|--------|---------|--------|
| Continue reading | Click button | Open reader at last position |
| View story | Click card | Navigate to story detail |
| Change status | Click status menu | Update library status |
| Remove | Click remove icon | Remove from library (confirm) |
| Move to collection | Drag or menu | Add to collection |

---

## 8. Bookmarks Page

### 8.1 Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  [← Library]                      My Bookmarks                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  🔍 Search bookmarks...                                         │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ The Dragon's Legacy                                         ││
│  │ Chapter 45: The Revelation                                  ││
│  │                                                              ││
│  │ "...she had heard a voice calling her name, beckoning       ││
│  │ her toward something she couldn't quite understand..."      ││
│  │                                                              ││
│  │ 📅 Added: Jan 15, 2024     📝 Note: "Great scene!"         ││
│  │                                                              ││
│  │ [Go to Chapter]                              [🗑️ Remove]    ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ The Secret Garden                                           ││
│  │ Chapter 12: The Discovery                                   ││
│  │                                                              ││
│  │ "...the door creaked open to reveal a garden unlike any     ││
│  │ she had ever seen, filled with flowers of every color..."   ││
│  │                                                              ││
│  │ 📅 Added: Jan 10, 2024                                      ││
│  │                                                              ││
│  │ [Go to Chapter]                              [🗑️ Remove]    ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Reading History Page

### 9.1 Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  [← Library]                    Reading History                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TODAY                                                           │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 2:30 PM   The Dragon's Legacy - Ch 45           12 min read ││
│  │ 1:15 PM   The Dragon's Legacy - Ch 44           15 min read ││
│  │ 10:00 AM  City of Stars - Ch 8                   8 min read ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  YESTERDAY                                                       │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 9:45 PM   The Secret Garden - Ch 36             20 min read ││
│  │ 8:30 PM   The Secret Garden - Ch 35             18 min read ││
│  │ 3:00 PM   The Dragon's Legacy - Ch 43           14 min read ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  JANUARY 13                                                      │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ...                                                         ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  [Load More History]                                             │
│                                                                  │
│  ────────────────────────────────────────────────────────────   │
│                                                                  │
│  [🗑️ Clear History]                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 10. Performance Optimizations

### 10.1 Content Virtualization

```tsx
// For long chapters (10K+ words)
function LongChapterContent({ content }) {
  return (
    <WindowScroller>
      {({ height, scrollTop }) => (
        <List
          autoHeight
          height={height}
          scrollTop={scrollTop}
          rowCount={paragraphs.length}
          rowHeight={({ index }) => measureParagraph(paragraphs[index])}
          rowRenderer={({ index, style }) => (
            <p style={style}>{paragraphs[index]}</p>
          )}
        />
      )}
    </WindowScroller>
  );
}
```

### 10.2 Prefetching

```typescript
// Prefetch next chapter while reading
function usePrefetchNextChapter(currentChapterId: string, nextChapterId?: string) {
  const queryClient = useQueryClient();
  
  useEffect(() => {
    if (nextChapterId) {
      queryClient.prefetchQuery({
        queryKey: ['chapter', nextChapterId],
        queryFn: () => api.chapters.get(nextChapterId),
        staleTime: 10 * 60 * 1000, // 10 minutes
      });
    }
  }, [nextChapterId, queryClient]);
}
```

### 10.3 Offline Reading

```typescript
// Cache chapters for offline reading
const { mutate: downloadChapter } = useMutation({
  mutationFn: async (chapterId: string) => {
    const chapter = await api.chapters.get(chapterId);
    await localDB.chapters.put(chapter);
    return chapter;
  },
});

// Download entire story for offline
async function downloadStoryForOffline(storyId: string) {
  const chapters = await api.stories.getChapters(storyId);
  await Promise.all(chapters.map(ch => downloadChapter(ch.id)));
}
```

---

## 11. Accessibility

### 11.1 Reader Accessibility

| Feature | Implementation |
|---------|----------------|
| Font scaling | System font size respected + custom controls |
| Dyslexia font | OpenDyslexic font option |
| High contrast | High contrast themes |
| Screen reader | Semantic HTML + ARIA |
| Reduced motion | Respect `prefers-reduced-motion` |
| Keyboard nav | Full keyboard support |

### 11.2 ARIA Labels

```tsx
<ReaderContent
  role="article"
  aria-label={`Chapter ${chapter.number}: ${chapter.title}`}
>
  <h1 aria-level={1}>{chapter.title}</h1>
  <div aria-live="polite" aria-atomic="true">
    Reading progress: {progress}%
  </div>
</ReaderContent>

<ProgressBar
  role="progressbar"
  aria-valuemin={0}
  aria-valuemax={100}
  aria-valuenow={progress}
  aria-label="Reading progress"
/>
```

---

## 12. References

### 12.1 Related Design Documents

| Document | Purpose |
|----------|----------|
| [03_STORY_PAGES.md](./03_STORY_PAGES.md) | Story detail |
| [02_DESIGN_SYSTEM_GUIDELINES.md](../02_DESIGN_SYSTEM_GUIDELINES.md) | UI components |
| [03_STATE_MANAGEMENT_ROUTING.md](../03_STATE_MANAGEMENT_ROUTING.md) | State patterns |
| [05_MOBILE_RESPONSIVE_DESIGN.md](../05_MOBILE_RESPONSIVE_DESIGN.md) | Mobile reading |

### 12.2 Backend Component References

| Document | Purpose |
|----------|----------|
| [05_READINGS_COMPONENT.md](../../../../Backend/N9/Documentation/BackendDesign/Components/05_READINGS_COMPONENT.md) | Reading progress, bookmarks, highlights |
| [01_STORIES_COMPONENT.md](../../../../Backend/N9/Documentation/BackendDesign/Components/01_STORIES_COMPONENT.md) | Chapters, content delivery |
| [03_PAYMENTS_COMPONENT.md](../../../../Backend/N9/Documentation/BackendDesign/Components/03_PAYMENTS_COMPONENT.md) | Premium chapter unlocking |

---

## 13. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|----------|
| 1.0 | 2025-12-31 | Frontend Team | Initial reader pages specification |
| 2.0 | 2026-01-04 | Frontend Team | Added comprehensive API endpoints (reading progress, bookmarks, highlights, library), offline reading enhancements, backend component references |
