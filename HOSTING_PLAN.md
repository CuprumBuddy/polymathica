# Hosting & Data Storage Plan for Polymathica

## Current State

- **Architecture**: Pure client-side app (HTML/CSS/JS)
- **Storage**: Browser localStorage only
- **Limitations**: No cross-device sync, no data backup, browser-specific

## Requirements

1. ✅ Host on GitHub Pages (or similar free service)
2. ✅ Data sync across devices
3. ✅ Only owner can edit data
4. ✅ Public read-only view with reduced details
5. ✅ Others can fork/clone to create their own version

---

## Recommended Approach: GitHub-Based Storage

### Architecture Overview

Use GitHub as both hosting platform and database:
- **GitHub Pages**: Host static site
- **GitHub API**: Store data in JSON file(s) in repo
- **GitHub OAuth**: Authenticate owner for write access
- **Public Access**: Anyone can view read-only version

### How It Works

```
┌─────────────────────────────────────────────────┐
│  Polymathica (GitHub Pages)                     │
│  https://username.github.io/learning-tracker/   │
└─────────────────────────────────────────────────┘
                    │
        ┌───────────┴───────────┐
        │                       │
    [Visitor]              [Owner (You)]
        │                       │
        │                   ┌───▼────────────┐
        │                   │ GitHub OAuth   │
        │                   │ Authentication │
        │                   └───┬────────────┘
        │                       │
        ▼                       ▼
┌───────────────┐      ┌────────────────┐
│ Read-Only     │      │ Full Edit Mode │
│ Public View   │      │ (Authenticated)│
└───────────────┘      └────────────────┘
        │                       │
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │ GitHub Repository     │
        │ data/user-data.json   │
        │ (version controlled)  │
        └───────────────────────┘
```

### Data Storage Strategy

#### File Structure
```
learning-tracker/
├── index.html
├── app.js
├── styles.css
├── auth.js              # NEW: GitHub OAuth handler
├── storage.js           # NEW: GitHub API storage adapter
├── data/
│   ├── subjects.js      # Catalog (read-only, public)
│   ├── summaries.js     # Summaries (read-only, public)
│   └── user-data.json   # NEW: User's personal data (synced via GitHub)
└── config.js            # NEW: Configuration (repo owner, etc.)
```

#### Data Separation

**Public Data (Anyone can read)**:
- `data/subjects.js` - Subject catalog structure
- `data/summaries.js` - Subject descriptions

**Private Data (Owner only, synced)**:
- `data/user-data.json`:
  ```json
  {
    "progress": {
      "subject-id": "partial",
      "subject-id-2": "complete"
    },
    "subjects": {
      "subject-id": {
        "goal": "User's personal goal",
        "resources": [...],
        "projects": [...],
        "notepad": "Private notes"
      }
    },
    "theme": "dark",
    "lastModified": "2025-11-29T12:00:00Z",
    "version": "2.0"
  }
  ```

**Public Read-Only View**:
- Shows subject catalog with summaries
- Hides personal goals, notes, resources, projects
- Progress shown as generic stats (e.g., "12/45 subjects completed")
- No edit buttons or modals

### Implementation Components

#### 1. GitHub OAuth Authentication

**Flow**:
1. User clicks "Sign in with GitHub"
2. Redirect to GitHub OAuth
3. Receive access token
4. Store token (sessionStorage or localStorage)
5. Verify token belongs to repo owner

**Code Structure**:
```javascript
// auth.js
class GitHubAuth {
  constructor(clientId, repoOwner) {
    this.clientId = clientId;
    this.repoOwner = repoOwner;
    this.token = null;
  }

  async login() {
    // OAuth flow
  }

  async verifyOwner() {
    // Check if authenticated user is repo owner
  }

  isAuthenticated() {
    return !!this.token;
  }

  isOwner() {
    return this.isAuthenticated() && this.username === this.repoOwner;
  }
}
```

#### 2. GitHub API Storage Adapter

**Operations**:
- Read user data
- Write user data (authenticated only)
- Handle merge conflicts
- Cache locally for offline access

**Code Structure**:
```javascript
// storage.js
class GitHubStorage {
  constructor(auth, repo, branch = 'main') {
    this.auth = auth;
    this.repo = repo; // "username/learning-tracker"
    this.branch = branch;
    this.apiBase = 'https://api.github.com';
  }

  async loadUserData() {
    // GET /repos/:owner/:repo/contents/data/user-data.json
    // Falls back to localStorage if offline
  }

  async saveUserData(data) {
    // Requires authentication
    // PUT /repos/:owner/:repo/contents/data/user-data.json
    // Also updates localStorage cache
  }

  async sync() {
    // Pull latest from GitHub
    // Merge with local changes
    // Push if owner is authenticated
  }
}
```

#### 3. View Mode System

**Modes**:
- `public`: Read-only, minimal details
- `authenticated-viewer`: Can see more but not edit
- `owner`: Full edit access

**UI Changes**:
```javascript
// app.js
function renderSubjectCard(subject, mode = 'public') {
  if (mode === 'public') {
    // Hide: goals, notepad, resources, projects
    // Show: name, description, prerequisites
    // No progress checkbox
  } else if (mode === 'owner') {
    // Show everything
    // Enable all editing features
  }
}
```

### Step-by-Step Implementation Plan

#### Phase 1: GitHub Pages Setup
1. Create GitHub repository (if not exists)
2. Enable GitHub Pages in repo settings
3. Configure custom domain (optional)
4. Test basic static site hosting

#### Phase 2: GitHub OAuth Setup
1. Register OAuth App in GitHub settings
   - Application name: "Polymathica"
   - Homepage URL: `https://username.github.io/learning-tracker/`
   - Callback URL: `https://username.github.io/learning-tracker/callback.html`
2. Store Client ID in `config.js`
3. Create `auth.js` for OAuth flow
4. Create `callback.html` to handle OAuth redirect

#### Phase 3: Storage Migration
1. Create `storage.js` - GitHub API adapter
2. Create `data/user-data.json` template
3. Migrate localStorage → GitHub API
4. Add sync button to UI
5. Implement auto-sync (every 5 minutes if changes)
6. Add conflict resolution (last-write-wins or manual)

#### Phase 4: View Modes
1. Add mode detection in `app.js`
2. Create public view (minimal details)
3. Add owner check after authentication
4. Hide/show UI elements based on mode
5. Add "Sign in to edit" prompt

#### Phase 5: Polish & Testing
1. Add loading states during sync
2. Add offline indicator
3. Test fork/clone workflow
4. Write documentation for forkers
5. Add export/import as backup option

---

## Alternative Approaches

### Option 2: Firebase/Supabase (Backend-as-a-Service)

**Pros**:
- Real-time sync out of the box
- Better performance for large datasets
- Built-in authentication
- Richer query capabilities

**Cons**:
- Additional service dependency
- API keys exposed in client code
- Less aligned with fork/clone workflow
- May have usage limits on free tier
- Data not in git (harder to fork)

**Best for**: If you need real-time collaboration or complex queries

---

### Option 3: Hybrid - Manual Sync

**Architecture**:
- Keep localStorage as primary
- Add export/import via GitHub Gist or repo file
- User manually triggers sync
- Simpler but less automatic

**Pros**:
- Simpler implementation
- No OAuth complexity
- Full offline support
- Easy to understand

**Cons**:
- Not automatic sync
- User must remember to sync
- Potential for data loss
- Less seamless experience

**Best for**: Minimal viable solution, quick implementation

---

### Option 4: Serverless Functions (Netlify/Vercel)

**Architecture**:
- Host on Netlify/Vercel (free tier)
- Serverless functions for backend
- Store data in Netlify Blobs or Vercel KV
- Use environment variables for secrets

**Pros**:
- Better security (API keys server-side)
- More control over backend logic
- Still free tier available
- Easy deployment

**Cons**:
- More complex than GitHub Pages
- Harder for others to fork (need their own deployment)
- Not purely static anymore
- Serverless cold starts

**Best for**: If you want more backend control

---

## Comparison Matrix

| Feature | GitHub API | Firebase | Hybrid Sync | Serverless |
|---------|-----------|----------|-------------|------------|
| **Free Forever** | ✅ Yes | ⚠️ Limits | ✅ Yes | ⚠️ Limits |
| **Auto Sync** | ✅ Yes | ✅ Yes | ❌ Manual | ✅ Yes |
| **Easy Fork** | ✅ Yes | ❌ No | ✅ Yes | ❌ Complex |
| **Offline First** | ✅ Yes | ⚠️ Partial | ✅ Yes | ⚠️ Partial |
| **No Extra Service** | ✅ Yes | ❌ No | ✅ Yes | ❌ No |
| **Version Control** | ✅ Yes | ❌ No | ✅ Yes | ❌ No |
| **Implementation** | 🟡 Medium | 🟢 Easy | 🟢 Easy | 🔴 Hard |
| **Real-time** | ❌ No | ✅ Yes | ❌ No | ✅ Yes |

---

## Recommended Timeline

### Week 1: Setup & Auth
- Day 1-2: GitHub Pages deployment
- Day 3-4: OAuth implementation
- Day 5-7: Test authentication flow

### Week 2: Storage Migration
- Day 1-3: Build storage adapter
- Day 4-5: Migrate localStorage to GitHub
- Day 6-7: Add sync functionality

### Week 3: View Modes
- Day 1-3: Implement public view
- Day 4-5: Add mode switching
- Day 6-7: UI polish

### Week 4: Testing & Docs
- Day 1-3: Test all scenarios
- Day 4-5: Write fork/clone guide
- Day 6-7: Deploy and monitor

---

## Public View Design

### What to Show (Read-Only)
- Subject catalog structure
- Subject names and categories
- Prerequisites/corequisites
- Summary descriptions (if available)
- Overall progress stats: "15 subjects in progress, 23 completed"

### What to Hide
- Personal goals
- Personal resources
- Projects
- Notepads
- Specific progress per subject
- Edit buttons
- Detail modals (or make them read-only)

### UI Mockup
```
┌────────────────────────────────────────┐
│ Polymathica           [Sign In to Edit]│
├────────────────────────────────────────┤
│ 📊 Public Learning Journey             │
│                                        │
│ 38 subjects explored                   │
│ 15 currently in progress               │
│ 23 completed                           │
│                                        │
│ ─── Mathematics ───                    │
│ □ Linear Algebra                       │
│   Foundation for advanced mathematics  │
│                                        │
│ □ Calculus I                           │
│   Differential and integral calculus   │
│                                        │
│ [Fork this tracker on GitHub]         │
└────────────────────────────────────────┘
```

---

## Security Considerations

### OAuth Token Storage
- Use `sessionStorage` for maximum security (clears on tab close)
- Never commit tokens to repo
- Add `.gitignore` for any token files
- Consider token expiration handling

### API Rate Limits
- GitHub API: 60 req/hour unauthenticated, 5000/hour authenticated
- Cache aggressively
- Batch updates
- Show rate limit status to user

### Data Privacy
- User data file is public in repo (by design)
- Don't store sensitive information
- Add warning in UI about public nature
- Consider optional private repo option

---

## Fork/Clone Workflow for Others

### For Someone Forking Your Tracker

**Instructions to add to README**:

```markdown
## Creating Your Own Tracker

1. **Fork this repository**
   - Click "Fork" button on GitHub
   - This creates your own copy

2. **Enable GitHub Pages**
   - Go to Settings → Pages
   - Source: Deploy from `main` branch
   - Your site: `https://yourusername.github.io/learning-tracker/`

3. **Set up OAuth (for syncing)**
   - Go to GitHub Settings → Developer settings → OAuth Apps
   - Create new OAuth App
   - Update `config.js` with your Client ID and username

4. **Customize your catalog**
   - Edit `data/subjects.js` to add/remove subjects
   - Edit `data/summaries.js` for descriptions

5. **Start tracking!**
   - Visit your GitHub Pages URL
   - Sign in with GitHub
   - Your data syncs automatically via GitHub
```

---

## Migration Path from Current State

### Step 1: Add Export Feature (NOW)
Even before implementing full GitHub sync, add export:
```javascript
function exportData() {
  const data = {
    subjects: JSON.parse(localStorage.getItem('subjects')),
    progress: JSON.parse(localStorage.getItem('subjectProgress')),
    theme: localStorage.getItem('theme')
  };

  const blob = new Blob([JSON.stringify(data, null, 2)],
    { type: 'application/json' });
  const url = URL.createObjectURL(blob);

  const a = document.createElement('a');
  a.href = url;
  a.download = `polymathica-backup-${Date.now()}.json`;
  a.click();
}
```

### Step 2: Deploy to GitHub Pages (NOW)
1. Push current code to GitHub
2. Enable Pages
3. Test that it works

### Step 3: Implement GitHub Storage (NEXT)
1. Add auth.js
2. Add storage.js
3. Migrate data structure

### Step 4: Add View Modes (THEN)
1. Detect authentication state
2. Render different views
3. Add public landing page

---

## Open Questions to Decide

1. **Public repo or private?**
   - Public: Anyone can see your data, easier to share
   - Private: More private, but limits public view feature

2. **Single JSON file or multiple?**
   - Single: Simpler, one file to manage
   - Multiple: Better organization, smaller commits

3. **Automatic sync frequency?**
   - On every change: Real-time but more API calls
   - Every 5 minutes: Balanced
   - Manual only: Most control

4. **Conflict resolution strategy?**
   - Last-write-wins: Simple but may lose data
   - Manual merge: More complex but safer
   - Show diff: Best UX but most work

5. **Fallback if GitHub is down?**
   - localStorage only: Always works offline
   - Show error: Simpler but less resilient

---

## Next Steps

1. **Decide**: Choose approach (recommend GitHub API)
2. **Setup**: Deploy current version to GitHub Pages
3. **Prototype**: Build auth + basic sync
4. **Test**: Verify cross-device sync works
5. **Polish**: Add public view and documentation
6. **Launch**: Share with others!

---

## Cost Analysis

### GitHub API Approach
- **Hosting**: Free (GitHub Pages)
- **Storage**: Free (within repo size limits)
- **Authentication**: Free (GitHub OAuth)
- **Bandwidth**: Free (reasonable use)
- **Total**: $0/month ✅

### Firebase Alternative
- **Hosting**: Free tier (10 GB/month)
- **Database**: Free tier (1 GB storage)
- **Authentication**: Free tier (no limit)
- **Can exceed free tier**: Yes (need monitoring)
- **Total**: $0-25/month ⚠️

### Serverless Alternative
- **Netlify/Vercel**: Free tier available
- **Functions**: 125k-100k requests/month free
- **Bandwidth**: 100 GB/month free
- **Can exceed free tier**: Yes
- **Total**: $0-20/month ⚠️

---

## Conclusion

**Recommended**: GitHub API approach
- ✅ Free forever
- ✅ Perfect for fork/clone workflow
- ✅ Version control built-in
- ✅ No additional services
- ✅ Aligns with current static architecture
- ⚠️ Requires OAuth implementation
- ⚠️ Manual conflict resolution needed

This approach keeps Polymathica simple, free, and forkable while adding the sync and sharing features you need.
