# JK BMS Monitor

A web-based Battery Management System monitor for real-time JK BMS device monitoring via Web Serial API.

## Features

- **Real-time Monitoring**: Live data updates from JK BMS devices
- **Cell Voltage Tracking**: Individual cell voltage monitoring and analysis
- **Battery Status Display**:
  - State of charge (SoC) percentage
  - Temperature monitoring
  - Current draw/charging rate
  - Voltage levels (pack and cell)
- **Serial Communication**: Direct connection via Web Serial API (USB)
- **Data Visualization**:
  - Cell voltage graphs
  - Temperature plots
  - Power flow diagrams
  - Performance trends
- **Temperature Monitoring**: Multi-sensor temperature tracking
- **Historical Data**: Track battery performance over time
- **Alerts and Warnings**: Notifications for out-of-range conditions
- **Offline Capable**: Store and review data when offline

## Tech Stack

- **HTML5** - Semantic markup
- **JavaScript** - BMS protocol implementation and data processing
- **Web Serial API** - Direct serial port communication
- **Canvas/SVG** - Data visualization and graphing
- **PWA** - Progressive Web App capabilities

## Getting Started

### Online Demo
Visit the live application: [https://arnoutzw.github.io/jk-bms-app/](https://arnoutzw.github.io/jk-bms-app/)

### Local Installation
Simply open `index.html` in a modern web browser. No build tools or dependencies required.

### Usage
1. Open the application in your browser
2. Connect your JK BMS device via USB cable
3. Click "Connect" and select your serial port from the browser dialog
4. Monitor real-time battery data:
   - Cell voltages
   - Pack voltage and current
   - Temperature readings
   - State of charge
5. View historical trends and performance data
6. Export data for analysis if needed

## Requirements

- JK BMS device with serial port communication capability
- Modern browser with Web Serial API support (Chrome/Edge on desktop)
- USB to serial adapter if JK BMS uses RS-485 or similar

## Browser Support

Web Serial API is supported in:
- Chrome 89+ (desktop)
- Edge 89+ (desktop)
- Opera 75+

Note: Firefox and Safari do not yet support the Web Serial API.

## License

MIT License - See LICENSE file for details
