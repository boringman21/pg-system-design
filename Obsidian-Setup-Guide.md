# 🧩 Obsidian Setup Guide - System Design Vault

**Tags**: #setup #guide #obsidian #configuration
**Date**: 2024-01-01

> Complete guide để optimize Obsidian experience cho System Design learning

## 🚀 Initial Setup

### **1. Installation**
1. Download [Obsidian](https://obsidian.md/) (free)
2. Install và open application
3. Click "Open folder as vault"
4. Select this repository folder
5. Click "Trust author and enable plugins" if prompted

### **2. First Launch**
- Obsidian sẽ open với [[Dashboard|🏠 Dashboard]] as homepage
- Left sidebar: File explorer, Search, Tags
- Right sidebar: Backlinks, Outline, Graph view
- Main pane: Content area

## 🔧 Essential Settings

### **File & Links** 
```
Settings > Files & Links:
✅ Use [[Wikilinks]]
✅ Automatically update internal links
✅ New link format: Relative path  
✅ Default location for new attachments: Assets/
✅ Default location for new notes: Current folder
```

### **Editor**
```
Settings > Editor:
✅ Live Preview
✅ Show line numbers
✅ Readable line length
✅ Show frontmatter
✅ Fold heading
✅ Fold indent
```

### **Appearance**
```
Settings > Appearance:
✅ Dark mode (recommended for long study sessions)
✅ Accent color: Blue (matches system design theme)
✅ Font size: 16px (readability)
```

## 📦 Recommended Plugins

### **Core Plugins** (Enable these)
- [ ] **Graph view** - Visualize knowledge connections
- [ ] **Backlinks** - See incoming references
- [ ] **Tag pane** - Browse by topics
- [ ] **Page preview** - Quick content preview
- [ ] **Search** - Full-text search
- [ ] **Templates** - Use standardized formats
- [ ] **Daily notes** - Track progress
- [ ] **Outline** - Navigate document structure

### **Community Plugins** (Install via Settings > Community plugins)

#### **Essential**
- [ ] **Templater** - Advanced templating
- [ ] **Tag Wrangler** - Manage tags efficiently
- [ ] **Calendar** - Visual progress tracking
- [ ] **Excalidraw** - Draw system diagrams

#### **Productivity**
- [ ] **Advanced Tables** - Better table editing
- [ ] **Dataview** - Query your notes
- [ ] **Quick Switcher++** - Enhanced navigation
- [ ] **Hotkeys++** - More keyboard shortcuts

#### **Visualization**
- [ ] **Mind Map** - Visual concept mapping
- [ ] **Journey** - Find connections between notes
- [ ] **Graph Analysis** - Advanced graph insights

## 🎨 Workspace Layout

### **Recommended Layout**
```
┌─────────────┬─────────────────────────┬─────────────┐
│ File        │                         │ Backlinks   │
│ Explorer    │                         │             │
├─────────────┤      Main Content       ├─────────────┤
│ Search      │                         │ Outline     │
├─────────────┤                         ├─────────────┤
│ Tags        │                         │ Graph View  │
└─────────────┴─────────────────────────┴─────────────┘
```

### **Pinned Tabs** (Always keep open)
- [[Dashboard|🏠 Dashboard]] - Main homepage
- [[MOC-System-Design|🗺️ Map of Content]] - Navigation hub  
- [[Quick-Reference|⚡ Quick Reference]] - Interview cheat sheet

## 🔍 Navigation Tips

### **Keyboard Shortcuts**
```
Ctrl/Cmd + O: Quick switcher (open any file)
Ctrl/Cmd + Shift + F: Global search
Ctrl/Cmd + G: Open graph view
Ctrl/Cmd + E: Toggle edit/preview mode
Ctrl/Cmd + [: Go back
Ctrl/Cmd + ]: Go forward
Ctrl/Cmd + Click: Open link in new tab
```

### **Search Techniques**
```
tag:#fundamental - Find all fundamental concepts
"exact phrase" - Search exact text
file:"CAP-Theorem" - Find specific file
path:"02-System-Components" - Search in specific folder
```

### **Graph View Usage**
- **Zoom**: Mouse wheel hoặc touchpad
- **Filter**: Use search box to show specific nodes
- **Groups**: Color-code by tags hoặc folders
- **Focus**: Click node to see connections

## 🏷️ Tag Strategy

### **Hierarchical Tags**
```
#fundamental/cap-theorem
#fundamental/acid-properties
#component/load-balancer
#component/cache
#case-study/url-shortener
#case-study/chat-system
```

### **Tag Combinations**
```
#fundamental #database - Database fundamentals
#performance #optimization - Performance optimization
#interview #practice - Interview practice materials
```

### **Tag Management**
- Use **Tag Wrangler** để rename/merge tags
- Keep tag hierarchy shallow (max 2 levels)
- Use consistent naming convention
- Regular cleanup to avoid tag bloat

## 📝 Note-Taking Workflow

### **Daily Study Session**
1. Open [[Dashboard|🏠 Dashboard]] to plan session
2. Navigate using [[MOC-System-Design|🗺️ Map of Content]]
3. Create notes using [[Templates/Learning-Notes-Template|📝 template]]
4. Link concepts với existing knowledge
5. Update progress on dashboard

### **Template Usage**
```
Ctrl/Cmd + T: Insert template
Choose appropriate template:
- Learning Notes: New concept study
- Case Study: System analysis
- System Design: Interview practice
```

### **Linking Strategy**
- Link **liberally** - more connections = better learning
- Use **descriptive link text**: `[[02-System-Components/Load-Balancers/Load-Balancer-Overview|Load Balancing]]`
- Create **bidirectional** connections
- Link to **templates** for consistency

## 📊 Progress Tracking

### **Daily Notes Setup**
```
Templates > Daily Note Template:
# {{title}}

## 📚 Today's Study
- [ ] Topic studied:
- [ ] Notes created:
- [ ] Concepts reviewed:

## 🎯 Key Insights
-

## 🔗 Connections Made
-

## 📝 Tomorrow's Plan
-
```

### **Weekly Review Process**
1. Review all notes created this week
2. Identify missing connections
3. Update [[Dashboard|🏠 Dashboard]] progress
4. Plan next week's focus areas

## 🎯 Advanced Features

### **Graph Analysis**
- **Orphaned notes**: Files with no links
- **Hub notes**: Files with many connections  
- **Clusters**: Related concept groups
- **Gaps**: Missing connections to explore

### **Dataview Queries** (if plugin installed)
```
List all fundamentals:
\```dataview
LIST FROM #fundamental

Recent case studies:
\```dataview
TABLE file.ctime as Created
FROM #case-study
SORT file.ctime DESC
\```

### **Canvas Usage** (Visual learners)
- Create visual **system diagrams**
- Map **concept relationships**
- Plan **study paths**
- Design **mock interview** flows

## 🔧 Troubleshooting

### **Common Issues**

**Links not working:**
- Check file path casing
- Ensure files exist at referenced location
- Use relative paths trong settings

**Graph view empty:**
- Enable all file types in graph settings
- Check if files have internal links
- Adjust filter settings

**Slow performance:**
- Close unused tabs
- Disable heavy plugins temporarily
- Index only necessary file types

**Missing backlinks:**
- Enable backlinks core plugin
- Check link format in settings
- Refresh vault (Ctrl/Cmd + R)

### **Performance Optimization**
```
Settings > Files & Links:
❌ Detect all file extensions (if large vault)
✅ Exclude files: *.tmp, *.log, node_modules

Settings > Appearance:
❌ Hardware acceleration (if laggy)
❌ Native menus (if buggy)
```

## 📚 Learning Resources

### **Obsidian Mastery**
- [Official Obsidian Help](https://help.obsidian.md/)
- [Obsidian Community](https://obsidian.md/community)
- YouTube: "Obsidian for Beginners"
- Course: "Linking Your Thinking" by Nick Milo

### **Note-Taking Methods**
- **Zettelkasten**: Permanent note system
- **PARA**: Projects, Areas, Resources, Archive
- **CODE**: Capture, Organize, Distill, Express
- **Building a Second Brain**: Capture và connect ideas

## ✅ Setup Checklist

### **Initial Setup**
- [ ] Obsidian installed và vault opened
- [ ] Essential core plugins enabled
- [ ] Community plugins installed
- [ ] Settings configured
- [ ] Workspace layout arranged

### **First Week**
- [ ] Daily notes template created
- [ ] First study session completed
- [ ] Links created between concepts
- [ ] Progress tracked on dashboard
- [ ] Graph view explored

### **Ongoing Optimization**
- [ ] Tag system refined
- [ ] Template library expanded
- [ ] Shortcuts memorized
- [ ] Workflow streamlined
- [ ] Regular reviews scheduled

---

*Obsidian is powerful knowledge management tool. Take time to learn its features - the investment sẽ pay dividends trong your System Design learning journey! 🚀*

## 🔗 Quick Links

**Main Navigation:** [[Dashboard|🏠 Dashboard]] | [[MOC-System-Design|🗺️ MOC]] | [[Index|📋 Index]]

**Templates:** [[Templates/Learning-Notes-Template|📝 Notes]] | [[Templates/System-Design-Template|🏗️ Design]] | [[Templates/Case-Study-Template|📖 Case Study]] 