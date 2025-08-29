# BTP Proxy - Secure AppRouter Architecture

## Overview
A secure SAP AppRouter application that provides SSH tunnel-only access to BTP destinations requiring Cloud Connector connectivity. This proxy enables local development against on-premises SAP systems while maintaining security.

## Architecture

### Data Flow
```
your-sapui5-app app:8080 → SSH tunnel:3000 → btp-proxy (AppRouter) → {SID} Destination → Cloud Connector → SAP System
```

### Security Model
- **No Public Access**: Public route removed via `cf unmap-route`
- **SSH Tunnel Only**: Access restricted to Cloud Foundry SSH tunneling
- **Pure AppRouter**: Standard SAP AppRouter with no custom middleware


## File Structure

```
btp-proxy/
├── package.json          # Dependencies and start script
├── xs-app.json           # Route configuration
├── manifest.yml          # CF deployment config
├── public/              # Static files
│   └── index.html
└── README.md      # This file
```

## Configuration Files

### package.json
```json
{
  "scripts": {
    "start": "node node_modules/@sap/approuter/approuter.js"
  },
  "dependencies": {
    "@sap/approuter": "^14.4.0",
    "@sap/xsenv": "^3.4.0"
  }
}
```

### xs-app.json
```json
{
  "welcomeFile": "/",
  "authenticationMethod": "none",
  "routes": [
    {
      "source": "^/sap/opu/odata/(.*)",
      "target": "/sap/opu/odata/$1",
      "destination": "{SID}",
      "authenticationType": "none",
      "csrfProtection": false
    },
    {
      "source": "^/(.*)",
      "target": "/$1",
      "localDir": "public",
      "authenticationType": "none"
    }
  ]
}
```

### manifest.yml
```yaml
---
applications:
- name: btp-proxy
  memory: 128M
  buildpacks:
    - nodejs_buildpack
  path: .
  services:
    - btp-proxy-dest
    - btp-proxy-conn
```

## Dependencies

### Required Dependencies
- **@sap/approuter**: Core SAP AppRouter functionality
- **@sap/xsenv**: BTP environment variable handling (VCAP_SERVICES)

## Service Bindings

### Required Services
- **btp-proxy-dest**: Destination service instance
- **btp-proxy-conn**: Connectivity service instance

### Service Creation Commands
```bash
# Create destination service instance
cf create-service destination lite btp-proxy-dest

# Create connectivity service instance  
cf create-service connectivity lite btp-proxy-conn

# Bind services via manifest.yml during push
```

## Deployment & Management

### Initial Deployment
```bash
# Deploy the proxy app
cf push

# Remove public route for security
cf unmap-route btp-proxy cfapps.eu10.hana.ondemand.com --hostname btp-proxy

# Enable SSH access
cf enable-ssh btp-proxy
cf restart btp-proxy
```

### Daily Operations
```bash
# Check app status
cf app btp-proxy

# View logs
cf logs btp-proxy --recent
cf logs btp-proxy

# Start/Stop/Restart
cf start btp-proxy
cf stop btp-proxy
cf restart btp-proxy
```

### SSH Tunnel Management
```bash
# Start SSH tunnel (run in separate terminal)
cf ssh btp-proxy -L 3000:localhost:8080 -N

# Stop SSH tunnel
# Press Ctrl+C in the tunnel terminal
```

## Local Development Workflow

### Step-by-Step Setup
1. **Deploy proxy to BTP**:
   ```bash
   cf push
   cf unmap-route btp-proxy cfapps.eu10.hana.ondemand.com --hostname btp-proxy
   ```

2. **Start SSH tunnel**:
   ```bash
   cf ssh btp-proxy -L 3000:localhost:8080 -N
   ```

3. **Run main application** (in separate terminal):
   ```bash
   cd pack-and-ship
   npm run start-local
   ```

4. **Access application**: http://localhost:8080

### Stopping Services
1. Stop main app: `Ctrl+C` in `npm run start-local` terminal
2. Stop SSH tunnel: `Ctrl+C` in `cf ssh` terminal

## Security Features

### Access Control
- **No Public Route**: Application not accessible from internet
- **SSH Authentication**: Requires CF login and SSH access
- **Space Isolation**: Only users with CF space access can tunnel

### Security Verification
```bash
# Verify external access is blocked
curl https://btp-proxy.cfapps.eu10.hana.ondemand.com/
# Should return: 404 Not Found

# Verify SSH tunnel works
curl http://localhost:3000/
# Should return: HTTP/1.1 302 Found
```

### Re-enabling Public Access (if needed)
```bash
# Add public route back
cf map-route btp-proxy cfapps.eu10.hana.ondemand.com --hostname btp-proxy
```

## Destination Configuration

### {SID} Destination Requirements
- **Name**: {SID}
- **Type**: HTTP  
- **ProxyType**: OnPremise
- **Authentication**: BasicAuthentication
- **CloudConnectorLocationId**: {SID}
- **URL**: Internal system URL

### Destination Service Flow
1. AppRouter requests destination "{SID}"
2. Destination service returns configuration + auth tokens
3. AppRouter uses Cloud Connector for OnPremise destinations
4. Cloud Connector routes to internal SAP system

## Troubleshooting

### Common Issues

#### "Route does not exist"
- **Cause**: Public route has been removed for security
- **Expected**: This confirms security is working
- **Solution**: Use SSH tunnel for access

#### "SSH connection failed"  
- **Check**: `cf app btp-proxy` (app must be running)
- **Fix**: `cf start btp-proxy`
- **Enable**: `cf enable-ssh btp-proxy && cf restart btp-proxy`

#### "Destination not found"
- **Check**: Service bindings in `cf app btp-proxy`
- **Verify**: Destination exists in BTP Cockpit
- **Debug**: `cf logs btp-proxy --recent`

#### "Cloud Connector error"
- **Message**: "Access denied to system connectivityproxy..."
- **Cause**: System not exposed in Cloud Connector
- **Solution**: Configure system exposure in SCC admin

### Health Checks
```bash
# Verify app is running
cf app btp-proxy

# Test SSH tunnel connectivity  
cf ssh btp-proxy -L 3000:localhost:8080 -N &
curl -I http://localhost:3000/

# Test main app integration
curl http://localhost:8080/sap/opu/odata/{partner_namespace}/{odata_service}/\$metadata
```

## Benefits of This Architecture

### Security
- **Zero external exposure**: App cannot be accessed from internet
- **CF authentication required**: Must be logged into CF with space access
- **SSH key security**: Leverages CF SSH key management

### Simplicity  
- **Pure AppRouter**: No custom code or complex middleware
- **Standard SAP patterns**: Follows official SAP documentation
- **Minimal dependencies**: Only essential SAP packages

### Reliability
- **Battle-tested**: Uses proven SAP AppRouter functionality
- **Low resource usage**: 128MB memory footprint
- **Easy debugging**: Standard AppRouter logging and troubleshooting

## Best Practices

### DO
- Use pure AppRouter with standard configuration
- Remove public routes for security
- Use SSH tunnels for local development
- Follow SAP destination service patterns
- Keep minimal dependencies

### DON'T  
- Add custom server.js files
- Create complex middleware
- Hardcode destination URLs
- Commit sensitive files (default-env.json)
- Test only in BTP - always test locally first

## References

- [SAP AppRouter Documentation](https://www.npmjs.com/package/@sap/approuter)
- [BTP Destination Service](https://help.sap.com/docs/CP_CONNECTIVITY/cca91383641e40ffbe03bdc78f00f681/7e306250e08340f89d6c103e28840f30.html)
- [Cloud Foundry SSH Access](https://docs.cloudfoundry.org/devguide/deploy-apps/ssh-apps.html)