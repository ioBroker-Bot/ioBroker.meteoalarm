# ioBroker Adapter Development with GitHub Copilot

**Version:** 0.4.0
**Template Source:** https://github.com/DrozmotiX/ioBroker-Copilot-Instructions

This file contains instructions and best practices for GitHub Copilot when working on ioBroker adapter development.

## Project Context

You are working on an ioBroker adapter. ioBroker is an integration platform for the Internet of Things, focused on building smart home and industrial IoT solutions. Adapters are plugins that connect ioBroker to external systems, devices, or services.

**[CUSTOMIZE: Meteoalarm Adapter Context]**

This is the **meteoalarm** adapter for ioBroker, which provides weather alarm data from European weather services. The adapter:

- Connects to various European meteorological services to fetch weather alerts and warnings
- Processes XML/JSON feeds containing weather alarm data (wind, rain, snow, temperature, etc.)
- Creates ioBroker states for different alarm levels (1-4) with icons, descriptions, and timing
- Supports geographic filtering using coordinates or region codes  
- Provides HTML formatted outputs for visualization in dashboards
- Uses external APIs and may require geographic coordinate validation
- Implements Sentry error reporting for production monitoring
- Processes polygon data for geographic boundary checking

Key technical aspects:
- Uses `xml2js` for XML parsing of weather data feeds
- Implements CSV parsing for geographic region mapping
- Uses `@turf/turf` for geographic polygon operations
- Integrates with `@iobroker/adapter-core` framework
- Supports both scheduled execution and on-demand data retrieval

## Testing

### Unit Testing
- Use Jest as the primary testing framework for ioBroker adapters
- Create tests for all adapter main functions and helper methods
- Test error handling scenarios and edge cases
- Mock external API calls and hardware dependencies
- For adapters connecting to APIs/devices not reachable by internet, provide example data files to allow testing of functionality without live connections
- Example test structure:
  ```javascript
  describe('AdapterName', () => {
    let adapter;
    
    beforeEach(() => {
      // Setup test adapter instance
    });
    
    test('should initialize correctly', () => {
      // Test adapter initialization
    });
  });
  ```

### Integration Testing

**IMPORTANT**: Use the official `@iobroker/testing` framework for all integration tests. This is the ONLY correct way to test ioBroker adapters.

**Official Documentation**: https://github.com/ioBroker/testing

#### Framework Structure
Integration tests MUST follow this exact pattern:

```javascript
const path = require('path');
const { tests } = require('@iobroker/testing');

// Define test coordinates or configuration
const TEST_COORDINATES = '52.520008,13.404954'; // Berlin
const wait = ms => new Promise(resolve => setTimeout(resolve, ms));

// Use tests.integration() with defineAdditionalTests
tests.integration(path.join(__dirname, '..'), {
    defineAdditionalTests({ suite }) {
        suite('Test adapter with specific configuration', (getHarness) => {
            let harness;

            before(() => {
                harness = getHarness();
            });

            it('should configure and start adapter', function () {
                return new Promise(async (resolve, reject) => {
                    try {
                        harness = getHarness();
                        
                        // Get adapter object using promisified pattern
                        const obj = await new Promise((res, rej) => {
                            harness.objects.getObject('system.adapter.your-adapter.0', (err, o) => {
                                if (err) return rej(err);
                                res(o);
                            });
                        });
                        
                        if (!obj) {
                            return reject(new Error('Adapter object not found'));
                        }

                        // Configure adapter properties
                        Object.assign(obj.native, {
                            position: TEST_COORDINATES,
                            createCurrently: true,
                            createHourly: true,
                            createDaily: true,
                            // Add other configuration as needed
                        });

                        // Set the updated configuration
                        harness.objects.setObject(obj._id, obj);

                        console.log('✅ Step 1: Configuration written, starting adapter...');
                        
                        // Start adapter and wait
                        await harness.startAdapterAndWait();
                        
                        console.log('✅ Step 2: Adapter started');

                        // Wait for adapter to process data
                        const waitMs = 15000;
                        await wait(waitMs);

                        console.log('🔍 Step 3: Checking states after adapter run...');
                        
                        // Check if states were created
                        const states = await harness.states.getKeysAsync('your-adapter.0.*');
                        console.log(`Found ${states.length} states`);
                        
                        // Verify specific expected states exist
                        expect(states.length).to.be.greaterThan(0);
                        
                        resolve();
                    } catch (error) {
                        console.error('❌ Test failed:', error.message);
                        reject(error);
                    }
                });
            });
        });
    }
});
```

#### Test Configuration Patterns

**Pattern 1: Basic Object Configuration**
```javascript
// Configure adapter settings through the adapter object
const obj = await harness.objects.getObjectAsync('system.adapter.your-adapter.0');
Object.assign(obj.native, {
    setting1: 'value1',
    setting2: true,
    coordinates: '52.520008,13.404954'
});
await harness.objects.setObjectAsync(obj._id, obj);
```

**Pattern 2: Direct Configuration Method**
```javascript
// Use the harness helper method for configuration
await harness.changeAdapterConfig('your-adapter', {
    native: {
        apiKey: 'test-key',
        interval: 300000,
        regions: ['DE', 'AT']
    }
});
```

#### State Validation Examples
```javascript
// Check if adapter created expected states
const connectionState = await harness.states.getStateAsync('your-adapter.0.info.connection');
expect(connectionState).to.not.be.null;
expect(connectionState.val).to.be.true;

// Validate data states exist and have expected structure
const dataKeys = await harness.states.getKeysAsync('your-adapter.0.alarms.*');
expect(dataKeys.length).to.be.greaterThan(0);

// Check state values
const levelState = await harness.states.getStateAsync('your-adapter.0.level');
expect(levelState.val).to.be.a('number');
expect(levelState.val).to.be.at.least(1).and.at.most(4);
```

#### Async Test Patterns
**Always** use proper timeout management for external API calls:
```javascript
it('should handle API calls within timeout', function () {
    // Set appropriate timeout for API operations
    this.timeout(120000); // 2 minutes for external APIs
    
    return new Promise(async (resolve, reject) => {
        try {
            // Test implementation
            resolve();
        } catch (error) {
            reject(error);
        }
    });
});
```

#### Mock External Dependencies
```javascript
// Mock external API calls in unit tests
const mockApiResponse = {
    alerts: [
        { level: 2, type: 'wind', description: 'Strong winds expected' }
    ]
};

// Use nock or similar for HTTP mocking
const nock = require('nock');
nock('https://api.example.com')
    .get('/weather/alerts')
    .reply(200, mockApiResponse);
```

**[CUSTOMIZE: Meteoalarm-Specific Testing Requirements]**

For the meteoalarm adapter, additional testing considerations:

```javascript
// Test geographic coordinate processing
const TEST_COORDINATES = '48.8566,2.3522'; // Paris coordinates

// Test configuration for European region
Object.assign(obj.native, {
    geocodeLocation: 'Paris, France',  
    geocode: TEST_COORDINATES,
    country: 'FR',
    provider: 1, // Meteoalarm provider
    imageSize: 50,
    noBackgroundColor: false,
    noIcons: false
});

// Validate weather alarm states
const alarmKeys = await harness.states.getKeysAsync('meteoalarm.0.alarms.*');
const htmlState = await harness.states.getStateAsync('meteoalarm.0.htmlToday');
const levelState = await harness.states.getStateAsync('meteoalarm.0.level');

// Check for proper HTML formatting
if (htmlState && htmlState.val) {
    expect(htmlState.val).to.include('<table');
    expect(htmlState.val).to.include('</table>');
}
```

## Adapter Development Best Practices

### Core Adapter Structure
Follow the standard ioBroker adapter pattern:

```javascript
const utils = require('@iobroker/adapter-core');
const adapter = new utils.Adapter('adapter-name');

// Essential event handlers
adapter.on('ready', () => {
    // Adapter startup logic
});

adapter.on('unload', (callback) => {
    // Cleanup resources, close connections
    callback();
});

// Optional: Handle object/state changes
adapter.on('stateChange', (id, state) => {
    // React to state changes
});
```

### Resource Management
Proper cleanup in the unload handler is crucial:

```javascript
adapter.on('unload', (callback) => {
  try {
    // Clear intervals and timeouts
    if (this.updateInterval) {
      clearInterval(this.updateInterval);
      this.updateInterval = undefined;
    }
    if (this.connectionTimer) {
      clearTimeout(this.connectionTimer);
      this.connectionTimer = undefined;
    }
    // Close connections, clean up resources
    callback();
  } catch (e) {
    callback();
  }
}
```

## Code Style and Standards

- Follow JavaScript/TypeScript best practices
- Use async/await for asynchronous operations
- Implement proper resource cleanup in `unload()` method
- Use semantic versioning for adapter releases
- Include proper JSDoc comments for public methods

## CI/CD and Testing Integration

### GitHub Actions for API Testing
For adapters with external API dependencies, implement separate CI/CD jobs:

```yaml
# Tests API connectivity with demo credentials (runs separately)
demo-api-tests:
  if: contains(github.event.head_commit.message, '[skip ci]') == false
  
  runs-on: ubuntu-22.04
  
  steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Use Node.js 20.x
      uses: actions/setup-node@v4
      with:
        node-version: 20.x
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run demo API tests
      run: npm run test:integration-demo
```

### CI/CD Best Practices
- Run credential tests separately from main test suite
- Use ubuntu-22.04 for consistency
- Don't make credential tests required for deployment
- Provide clear failure messages for API connectivity issues
- Use appropriate timeouts for external API calls (120+ seconds)

### Package.json Script Integration
Add dedicated script for credential testing:
```json
{
  "scripts": {
    "test:integration-demo": "mocha test/integration-demo --exit"
  }
}
```

### Practical Example: Complete API Testing Implementation
Here's a complete example based on lessons learned from the Discovergy adapter:

#### test/integration-demo.js
```javascript
const path = require("path");
const { tests } = require("@iobroker/testing");

// Helper function to encrypt password using ioBroker's encryption method
async function encryptPassword(harness, password) {
    const systemConfig = await harness.objects.getObjectAsync("system.config");
    
    if (!systemConfig || !systemConfig.native || !systemConfig.native.secret) {
        throw new Error("Could not retrieve system secret for password encryption");
    }
    
    const secret = systemConfig.native.secret;
    let result = '';
    for (let i = 0; i < password.length; ++i) {
        result += String.fromCharCode(secret[i % secret.length].charCodeAt(0) ^ password.charCodeAt(i));
    }
    
    return result;
}

// Run integration tests with demo credentials
tests.integration(path.join(__dirname, ".."), {
    defineAdditionalTests({ suite }) {
        suite("API Testing with Demo Credentials", (getHarness) => {
            let harness;
            
            before(() => {
                harness = getHarness();
            });

            it("Should connect to API and initialize with demo credentials", async () => {
                console.log("Setting up demo credentials...");
                
                if (harness.isAdapterRunning()) {
                    await harness.stopAdapter();
                }
                
                const encryptedPassword = await encryptPassword(harness, "demo_password");
                
                await harness.changeAdapterConfig("your-adapter", {
                    native: {
                        username: "demo@provider.com",
                        password: encryptedPassword,
                        // other config options
                    }
                });

                console.log("Starting adapter with demo credentials...");
                await harness.startAdapter();
                
                // Wait for API calls and initialization
                await new Promise(resolve => setTimeout(resolve, 60000));
                
                const connectionState = await harness.states.getStateAsync("your-adapter.0.info.connection");
                
                if (connectionState && connectionState.val === true) {
                    console.log("✅ SUCCESS: API connection established");
                    return true;
                } else {
                    throw new Error("API Test Failed: Expected API connection to be established with demo credentials. " +
                        "Check logs above for specific API errors (DNS resolution, 401 Unauthorized, network issues, etc.)");
                }
            }).timeout(120000);
        });
    }
});
```

**[CUSTOMIZE: Meteoalarm-Specific Development Patterns]**

### Weather Data Processing
The meteoalarm adapter follows specific patterns for processing weather alarm data:

```javascript
// Parse weather alarm XML/JSON data
async function parseWeatherData(xmlData) {
    const parser = new xml2js.Parser();
    const result = await parser.parseStringPromise(xmlData);
    
    // Extract alarm information
    const alarms = result.alerts?.alert || [];
    return alarms.map(alarm => ({
        identifier: alarm.identifier?.[0],
        level: parseInt(alarm.severity?.[0]),
        type: alarm.event?.[0],
        description: alarm.description?.[0],
        effective: alarm.effective?.[0],
        expires: alarm.expires?.[0],
        areas: alarm.area || []
    }));
}

// Geographic boundary checking
function checkIfInPoly(coordinates, polygonData) {
    const point = turf.point(coordinates);
    const polygon = turf.polygon(polygonData);
    return turf.booleanPointInPolygon(point, polygon);
}

// HTML output generation
function generateAlarmHTML(alarms, config) {
    let html = '<table><tbody>';
    
    for (const alarm of alarms) {
        const colorStyle = config.noBackgroundColor ? '' : `background-color: ${getAlarmColor(alarm.level)}`;
        const iconImg = config.noIcons ? '' : `<img src="${getAlarmIcon(alarm.type)}" style="height: ${config.imageSize}px;">`;
        
        html += `<tr>
            <td style="${colorStyle}">${iconImg}</td>
            <td style="${colorStyle}">${alarm.description}</td>
        </tr>`;
    }
    
    html += '</tbody></table>';
    return html;
}
```

### Configuration Validation
```javascript
// Validate geographic coordinates
function validateCoordinates(coordString) {
    const match = coordString.match(/^(-?\d+\.?\d*),(-?\d+\.?\d*)$/);
    if (!match) return false;
    
    const lat = parseFloat(match[1]);
    const lon = parseFloat(match[2]);
    
    return lat >= -90 && lat <= 90 && lon >= -180 && lon <= 180;
}

// Validate region codes
function validateRegionCode(code) {
    return typeof code === 'string' && /^[A-Z]{2}$/.test(code);
}
```