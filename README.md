# Sound Object Phenomenon - Research Tool
**UCI Hearing & Speech Lab**

## 📊 Overview

The Sound Object Phenomenon tool is a Progressive Web App (PWA) designed for acoustic perception research. It enables researchers to study how people visualize sound frequencies by allowing participants to draw shapes in response to different frequency stimuli. The tool automatically calculates geometric properties of drawn shapes including area and centroid coordinates, providing quantitative data for analyzing sound-shape associations.

### Key Features
- **Multi-frequency testing**: 11 predefined frequencies (31 Hz to 16,000 Hz) at various dB levels
- **Drawing interface**: Touch and mouse-compatible canvas with adjustable brush sizes and 7 color options
- **Real-time analysis**: Automatic calculation of shape area and centroid position using standardized units
- **Data export**: CSV files and direct Google Sheets integration for centralized data collection
- **Offline capability**: Works without internet after initial installation (PWA technology)
- **Background images**: Support for reference images during drawing tasks
- **Undo/redo functionality**: Full drawing history for each frequency
- **Reset capability**: Quick reset between participants while preserving reference images

### Research Applications
- Sound perception studies
- Cross-modal correspondence research
- Auditory-visual synesthesia investigation
- Frequency-shape mapping analysis
- Psychoacoustic research

---

## 🚀 Quick Start Installation Guide

### For Researchers (Initial Setup)

#### Option 1: GitHub Pages (Recommended - Free & Easy)
1. **Create a GitHub account** at https://github.com (free)
2. **Create a new repository**
   - Click "New repository"
   - Name it `sound-object-tool`
   - Set to Public
   - Initialize with README
3. **Upload the files**
   - Click "Upload files"
   - Upload all 5 files (see required files below)
   - Commit changes
4. **Enable GitHub Pages**
   - Go to Settings → Pages
   - Source: Deploy from branch
   - Branch: main / root
   - Save
5. **Your app URL will be**: `https://[yourusername].github.io/sound-object-tool/`

#### Option 2: Institutional Web Server
1. Request web space from your IT department
2. Upload all files via FTP/SFTP
3. Ensure HTTPS is enabled
4. Share the URL with colleagues

### For Participants (Installing on Tablets)

#### iPad/iPhone:
1. Open **Safari** browser
2. Go to the provided URL
3. Tap the **Share** button (square with arrow)
4. Tap **"Add to Home Screen"**
5. Name it "Sound Object" and tap **Add**

#### Android:
1. Open **Chrome** browser
2. Go to the provided URL
3. Tap menu (3 dots) → **"Add to Home screen"**
4. Tap **Install**

#### Windows/Surface:
1. Open **Edge** browser
2. Go to the provided URL
3. Click menu → Apps → **"Install this site as an app"**

---

## 📁 Required Files

Before deployment, ensure you have these 5 files:

1. **sound_visualization_tool_FINAL.html** - Main application
2. **manifest.json** - PWA configuration
3. **service-worker.js** - Offline functionality
4. **icon-192.png** - Small app icon (convert from SVG)
5. **icon-512.png** - Large app icon (convert from SVG)

### Converting Icons (Required)
The provided icons are in SVG format and must be converted to PNG:
- Use online converter: https://cloudconvert.com/svg-to-png
- Or ask IT department for assistance
- Ensure filenames are exactly: `icon-192.png` and `icon-512.png`

---

## ✅ Deployment Checklist

### Pre-Deployment
- [ ] Convert icon-192.svg → icon-192.png
- [ ] Convert icon-512.svg → icon-512.png
- [ ] Test locally by opening HTML file in browser
- [ ] Verify all buttons and features work
- [ ] Choose hosting method (GitHub Pages or institutional server)

### GitHub Pages Deployment
- [ ] Create GitHub account
- [ ] Create new public repository
- [ ] Upload all 5 files
- [ ] Enable GitHub Pages in Settings
- [ ] Wait 5-10 minutes for deployment
- [ ] Test URL: `https://[yourusername].github.io/[repository-name]/`

### Post-Deployment Testing
- [ ] Visit URL on computer - verify it loads
- [ ] Test on iPad/tablet - verify touch drawing works
- [ ] Install as PWA - verify home screen icon appears
- [ ] Test offline mode - turn off WiFi and verify app still works
- [ ] Test data export - verify CSV download works
- [ ] Share URL with 1-2 colleagues for testing

### Google Sheets Integration (Optional)
- [ ] Create Google Sheet for data collection
- [ ] Set up Google Apps Script (see separate guide)
- [ ] Test data export to sheets
- [ ] Save script URL in app

---

## 🔧 Technical Specifications

### Coordinate System
- **Grid**: 20×20 units centered at origin (0,0)
- **Range**: -10 to +10 on both X and Y axes
- **Scale**: 1 unit = 50 pixels (on 1000×1000px canvas)
- **Area**: Measured in square units (1 unit² = 2,500 pixels²)
- **Calculation**: Shoelace formula for polygon areas

### Supported Frequencies
| Frequency (Hz) | SPL (dB) |
|---------------|----------|
| 31            | 100      |
| 62.5          | 100      |
| 125           | 90       |
| 250           | 85       |
| 500           | 80       |
| 1000          | 80       |
| 2000          | 80       |
| 4000          | 80       |
| 8000          | 90       |
| 12000         | 85       |
| 16000         | 85       |

### Browser Compatibility
- **Recommended**: Safari (iOS), Chrome (Android/Windows)
- **Supported**: Edge, Firefox
- **Features**: Touch events, PWA installation, local storage

---

## 📝 Usage Instructions

### Running a Study Session

1. **Setup**
   - Open app on tablet
   - Enter participant ID (e.g., P-001)
   - Upload reference image if needed

2. **For Each Frequency**
   - Select frequency tab
   - Play corresponding sound
   - Participant draws perceived shape
   - Multiple shapes/colors allowed per frequency

3. **Data Collection**
   - Review area and coordinate data in real-time
   - Export CSV for individual participant
   - Or send to Google Sheets for centralized collection

4. **Between Participants**
   - Click "Reset All" button
   - Confirms preservation of background image
   - All drawings and settings cleared
   - Ready for next participant

### Tips for Researchers
- Test the tool yourself before participant sessions
- Ensure quiet environment for testing
- Use consistent sound playback equipment
- Consider randomizing frequency presentation order
- Export data after each participant as backup

---

## 🛟 Troubleshooting

### App Won't Install
- Ensure using correct browser (Safari on iOS, Chrome on Android)
- Clear browser cache and retry
- Verify all files uploaded correctly
- Check HTTPS is enabled (required for PWA)

### Drawing Not Working
- Check "Stop/Start Drawing" toggle
- Ensure JavaScript is enabled
- Try refreshing the page
- Test with different browser

### Export Issues
- CSV: Check browser download settings
- Google Sheets: Verify script URL is correct
- Ensure internet connection for sheets export

### Icons Not Showing
- Verify PNG conversion completed
- Check filenames exactly match (icon-192.png, icon-512.png)
- Ensure icons in same folder as HTML

---

## 🔒 Privacy & Data

- All drawing data stored locally on device
- No automatic data transmission
- Participant controls all data export
- No personal information collected beyond participant ID
- Compliant with research data protocols

---

## 📄 License & Citation

This tool was developed by the UCI Hearing & Speech Lab for acoustic perception research.
```

---

## Version History

- **v1.0** (2024) - Initial release with PWA support
  - Multi-frequency drawing interface
  - Real-time geometric analysis
  - CSV and Google Sheets export
  - Offline functionality

---

*Last updated: 2024*
