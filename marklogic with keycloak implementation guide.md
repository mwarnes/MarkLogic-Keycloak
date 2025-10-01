# MarkLogic with Keycloak Implementation Guide

**Author**: Martin Warnes, Principal Technical Support Engineer, Progress Software

## Introduction

This document provides comprehensive methods and procedures for implementing Keycloak as an identity provider for MarkLogic Application Servers. The guide covers the configuration and integration of Keycloak and MarkLogic to support both OAuth2 and SAML authentication protocols, enabling secure single sign-on (SSO) capabilities for MarkLogic applications. By following the procedures outlined in this document, organizations can leverage Keycloak's robust identity and access management features to enhance security, streamline user authentication, and provide a seamless user experience across their MarkLogic-based applications.

### About MarkLogic

MarkLogic is an enterprise NoSQL database platform originally developed by MarkLogic Corporation and now owned by Progress Software. It is designed to handle complex, multi-structured data including documents, metadata, collections, and semantic triples. MarkLogic provides a unified platform for data integration, search, analytics, and application development, making it particularly suitable for content-driven applications and enterprise data hub architectures. The platform includes built-in search capabilities, ACID transactions, and enterprise-grade security features.

**Website**: [https://www.progress.com/marklogic](https://www.progress.com/marklogic)

### About Keycloak

Keycloak is an open-source identity and access management solution developed by Red Hat. It provides comprehensive authentication and authorization services for applications and services, supporting industry-standard protocols including OAuth 2.0, OpenID Connect, and SAML 2.0. Keycloak offers features such as single sign-on (SSO), identity brokering, user federation, fine-grained authorization policies, and social login capabilities. As a Red Hat project, it benefits from enterprise support and continuous development by a vibrant open-source community.

**Website**: [https://www.keycloak.org](https://www.keycloak.org)

## Scope and Assumptions

This guide concentrates specifically on creating a Keycloak client for OAuth2 authentication with MarkLogic Application Servers. The following assumptions are made:

- Keycloak has already been installed and is running
- A Keycloak realm is already available and configured
- Keycloak users and roles for MarkLogic have been configured
- Basic familiarity with MarkLogic administration
- Administrative access to both Keycloak and MarkLogic systems

The document will focus on the client configuration process rather than the initial Keycloak installation or realm setup procedures.

## Creating a Keycloak OpenID Connect Client

Create a Keycloak client for use by MarkLogic, the client should use "Standard Flow" Authentication. 

**Note:** MarkLogic does not support "Implicit Flow" so the Access settings can be left unpopulated.

### Keycloak Client Configuration Settings

The following screenshot shows an example configuration for a Keycloak client:

![Keycloak Client Configuration Settings](images/Keycloak%20Client%20configuration.png)

### Keycloak Roles Considerations

#### JSON Object Array Compatibility Issue

Keycloak returns user roles as JSON object arrays within the `realm_access` and `resource_access` sections of the OpenID access token. However, MarkLogic cannot directly process these nested JSON object arrays for role extraction, which creates a compatibility challenge.

#### OpenID Access Token Structure

A typical Keycloak OpenID access token contains roles in the following structure:

```json
{
  "exp": 1759152574,
  "iat": 1759152274,
  "iss": "https://oauth.warnesnet.com:8443/realms/progress-marklogic",
  "realm_access": {
    "roles": ["marklogic-user", "marklogic-manage", "marklogic-admin"]
  },
  "resource_access": {
    "account": {
      "roles": ["manage-account", "view-profile"]
    }
  },
  "preferred_username": "martin",
  "email": "martin@example.com"
}
```

#### The Problem

- **`realm_access.roles`**: Contains an array of realm-level roles nested within a JSON object
- **`resource_access.[client].roles`**: Contains client-specific roles in a similar nested structure
- **MarkLogic Limitation**: MarkLogic's OAuth2 implementation cannot parse roles from these nested JSON object arrays

#### The Solution

To work around this limitation, Keycloak must be configured to provide roles in a format that MarkLogic can process. This typically involves:

1. **Custom Claim Mapping**: Creating a custom mapper that flattens the roles into a simple array
2. **Direct Role Claims**: Configuring a top-level claim (like `marklogic-roles` in the example above) that contains roles as a simple string array
3. **Protocol Mappers**: Using Keycloak's built-in mappers to transform the token structure

The following example demonstrates a Protocol Mapper that has been added to a Profile Scope, which extracts user Realm roles and returns them as a new claim using a string data type and name "marklogic-roles".

![Keycloak Mapper](images/Keycloak%20Mapper.png)

MarkLogic will be able to access these roles by referencing the claim in the External Security profile configuration.

```json
{
  "exp": 1759152574,
  "iat": 1759152274,
  "iss": "https://oauth.warnesnet.com:8443/realms/progress-marklogic",
  "realm_access": {
    "roles": ["marklogic-user", "marklogic-manage", "marklogic-admin"]
  },
  "resource_access": {
    "account": {
      "roles": ["manage-account", "view-profile"]
    }
  },
  "marklogic-roles": ["marklogic-user", "marklogic-manage", "marklogic-admin"],
  "preferred_username": "martin",
  "email": "martin@example.com"
}
```

## Creating a MarkLogic External Security Profile

Once the Keycloak client is configured with the appropriate role mappers, the next step is to create an External Security profile in MarkLogic that will integrate with the Keycloak identity provider.

### Obtaining the JWKS URI from Keycloak

Before configuring the MarkLogic External Security profile, you need to obtain the JWKS (JSON Web Key Set) URI from your Keycloak realm. This URI is required by MarkLogic to validate the JWT tokens issued by Keycloak.

#### Steps to Find the JWKS URI:

1. **Access Keycloak Admin Console**: Log into your Keycloak admin interface
2. **Navigate to Realm Settings**: Select your realm (e.g., "progress-marklogic")
3. **Find OpenID Endpoint Configuration**: In the Realm Settings, look for the "OpenID Endpoint Configuration" link
4. **Access the Configuration**: Click the link to view the OpenID configuration JSON
5. **Locate JWKS URI**: In the configuration JSON, find the `jwks_uri` field

The JWKS URI typically follows this format:
```
https://your-keycloak-server:port/realms/your-realm-name/protocol/openid-connect/certs
```
### Configuring the MarkLogic External Security Profile

Access the MarkLogic AdminUI interface and navigate to the External Security configuration page. 

#### Required Configuration Settings:

1. **External Security Name**: Provide a descriptive name (e.g., "Keycloak-OAuth2")

2. **Authentication Protocol**: Select "oauth2"

3. **Cache Timeout**: Set according to your security requirements (typically 300-3600 seconds)

4. **Authorization**: Select "oauth2"

5.  **OAuth2 Settings**:
    - **OAuth Flow Type**: Select "Resource Server"
    - **OAuth Vendor**: Select "Other"
    - **OAuth Client ID**: The client id from your Keycloak client configuration
    - **OAuth Token Type**: Select "JSON Web Tokens"
    - **OAuth username attribute**: Enter the claim holding the userid from your Keycloak client configuration
    - **OAuth role attribute**; Enter "marklogic-roles" (or the custom claim name you configured in Keycloak) to map Keycloak roles to MarkLogic roles as needed
    - **OAuth JWT Algorithm**: Select "RS256"
    - **JWKS URI**: The JWKS URI obtained from the Keycloak OpenID configuration

### Example Configuration Reference

For reference, the following is an example of a configured External Security profile:

![MarkLogic OAUTH External Security Configuration](images/MarkLogic%20OAUTH%20External%20Security%20Configuration.png)


### Testing the Configuration

After creating the External Security profile, you can test the OAuth2 integration using various methods.

#### Manual Token Testing with curl

You can test the Keycloak token generation and MarkLogic integration using the following curl command to obtain a JWT token directly from Keycloak:

```bash
curl --location 'https://oauth.warnesnet.com:8443/realms/progress-marklogic/protocol/openid-connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'username=martin' \
--data-urlencode 'password=sterling123' \
--data-urlencode 'client_id=marklogic-oauth' \
--data-urlencode 'client_secret=4UZyJkjWsGV5JtpsWfgkL1qW5vZ5hhmv' \
--data-urlencode 'grant_type=password' \
--data-urlencode 'scope=openid'
```

#### Parameter Explanations

- **URL**: `https://oauth.warnesnet.com:8443/realms/progress-marklogic/protocol/openid-connect/token`
  - The Keycloak token endpoint for your specific realm
- **Content-Type**: `application/x-www-form-urlencoded`
  - Required header for OAuth2 token requests
- **username**: `martin`
  - The Keycloak user account to authenticate
- **password**: `sterling123`
  - The user's password (Note: This is the Resource Owner Password Credentials grant type)
- **client_id**: `marklogic-oauth`
  - The client ID configured in Keycloak for MarkLogic
- **client_secret**: `4UZyJkjWsGV5JtpsWfgkL1qW5vZ5hhmv`
  - The client secret from your Keycloak client configuration
- **grant_type**: `password`
  - OAuth2 grant type (Resource Owner Password Credentials)
- **scope**: `openid`
  - Requested OAuth2 scope to include OpenID Connect claims

#### Extracting and Storing the Access Token

To extract the access token and store it in an environment variable for subsequent API calls, use the following curl command as an example:

```bash
# Extract access token and store in environment variable
ACCESS_TOKEN=$(curl --silent --location -k 'https://oauth.warnesnet.com:8443/realms/progress-marklogic/protocol/openid-connect/token' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'username=martin' \
--data-urlencode 'password=sterling123' \
--data-urlencode 'client_id=marklogic-oauth' \
--data-urlencode 'client_secret=4UZyJkjWsGV5JtpsWfgkL1qW5vZ5hhmv' \
--data-urlencode 'grant_type=password' \
--data-urlencode 'scope=openid' | jq -r '.access_token')

# Verify the token was extracted
echo "Access Token: $ACCESS_TOKEN"

# Export for use in subsequent commands
export ACCESS_TOKEN
```

**Prerequisites for token extraction:**
- `jq` command-line JSON processor must be installed
- The curl command must receive a successful response from Keycloak

#### Using the Token with MarkLogic

Once you have the access token, you can test it against a MarkLogic endpoint:

```bash
# Test the token against a MarkLogic endpoint
curl --location 'http://oauth.warnesnet.com:8002/manage/LATEST/' \
--header "Authorization: Bearer $ACCESS_TOKEN"
```

#### Verification Steps

After obtaining and using the token:

1. **Save the Configuration**: Ensure all settings are saved in MarkLogic
2. **Apply to App Server**: Associate the External Security profile with your target App Server
3. **Verify Role Mapping**: Confirm that roles from Keycloak are properly mapped in MarkLogic via the Roles External Name field
4. **Test Token Generation**: Use the curl command above to obtain a valid JWT token
5. **Test MarkLogic Authentication**: Use the token to authenticate against MarkLogic endpoints
6. **Check Logs**: Review MarkLogic error logs for any authentication issues
7. **Validate Role Assignment**: Verify that the user receives appropriate roles and permissions in MarkLogic

**Security Note**: The Resource Owner Password Credentials grant type should only be used for testing purposes. In production environments, use the Authorization Code flow in your application for better security.

## OAuth2 Troubleshooting

When OAuth2 authentication between Keycloak and MarkLogic isn't working as expected, systematic troubleshooting of both systems is essential. This section provides guidance on reviewing logs and enabling debug traces to identify and resolve integration issues.

### MarkLogic Troubleshooting

#### Enabling Debug Trace Events

To increase the level of debug messages in MarkLogic logs, enable the following trace event:

1. **Access MarkLogic Admin Console**: Navigate to `http://your-marklogic-server:8001`
2. **Go to Groups**: Select "Configure" > "Groups" > "Default"
3. **Navigate to Diagnostics**: Click on "Diagnostics" in the left navigation
4. **Add Trace Event**: In the "Trace Events" section, add the following trace events:
   - `JSON Status`
   - `JSON Communications`
   - `JSON Processing`

This trace event will provide detailed information about JSON processing and OAuth2 token validation in the MarkLogic logs.

#### Key MarkLogic Log Messages to Look For

When troubleshooting OAuth2 issues, watch for these types of log entries:

- **Token Validation Errors**: Messages about JWT signature verification failures
- **JWKS URI Issues**: Problems accessing or parsing the Keycloak JWKS endpoint
- **Role Mapping Problems**: Issues with extracting or mapping roles from tokens
- **JSON Processing Issues**: Token parsing or claim extraction failures

#### Common MarkLogic OAuth2 Error Patterns

```
# JWT signature validation failure
XDMP-OAUTH2: Invalid JWT signature

# JWKS URI access issues
XDMP-OAUTH2: Unable to retrieve JWKS from URI

# Role claim not found
XDMP-OAUTH2: Role attribute not found in token

# Token expiration
XDMP-OAUTH2: JWT token has expired
```

### Keycloak Troubleshooting

#### Keycloak Log Locations

Keycloak logs are typically located in:

**Standalone Mode:**
```bash
# Server logs
$KEYCLOAK_HOME/standalone/log/server.log

# Access logs (if enabled)
$KEYCLOAK_HOME/standalone/log/access.log
```

**Docker Deployments:**
```bash
# View container logs
docker logs keycloak-container-name

# Follow logs in real-time
docker logs -f keycloak-container-name
```

#### Enabling Debug Logging in Keycloak

To enable more detailed logging in Keycloak:

1. **Access Admin Console**: Log into Keycloak admin interface
2. **Navigate to Events**: Go to "Realm Settings" > "Events"
3. **Enable Event Logging**: 
   - Turn on "Save Events"
   - Turn on "Save Admin Events"
   - Set appropriate expiration times
4. **Configure Log Levels** (via CLI or configuration):

```bash
# Enable debug logging for OAuth2/OIDC
/subsystem=logging/logger=org.keycloak.protocol.oidc:add(level=DEBUG)
/subsystem=logging/logger=org.keycloak.services.resources.LoginActionsService:add(level=DEBUG)
/subsystem=logging/logger=org.keycloak.authentication:add(level=DEBUG)
```

#### Key Keycloak Log Messages to Monitor

- **Authentication Events**: User login attempts and results
- **Token Generation**: OAuth2/OIDC token creation and signing
- **Client Authentication**: Client credential validation
- **Mapper Execution**: Protocol mapper processing and claim generation
- **CORS Issues**: Cross-origin request problems
- **Client Configuration Errors**: Invalid redirect URIs or client settings

### Common OAuth2 Integration Issues

#### 1. Token Signature Verification Failures

**Symptoms:**
- MarkLogic rejects valid tokens
- "Invalid JWT signature" errors in MarkLogic logs

**Solutions:**
- Verify JWKS URI is accessible from MarkLogic server
- Check network connectivity and firewall rules
- Ensure MarkLogic system time is synchronized
- Validate the JWKS URI in External Security profile

#### 2. Role Mapping Problems

**Symptoms:**
- Users authenticate but receive insufficient permissions
- Role-related errors in MarkLogic logs

**Solutions:**
- Verify the custom role claim name in both Keycloak mapper and MarkLogic profile
- Check that the Protocol Mapper is correctly configured and enabled
- Ensure the claim appears in the token (decode JWT to verify)
- Validate MarkLogic role mapping configuration

#### 3. Client Configuration Mismatches

**Symptoms:**
- Authentication flows fail at various stages
- Client authentication errors in Keycloak logs

**Solutions:**
- Verify client ID matches between Keycloak and MarkLogic
- Check client secret if using confidential client
- Ensure redirect URIs are correctly configured
- Validate OAuth2 flow settings (Standard Flow enabled)

#### 4. Network and Connectivity Issues

**Symptoms:**
- Intermittent authentication failures
- Timeout errors in logs

**Solutions:**
- Test network connectivity between MarkLogic and Keycloak
- Check DNS resolution for Keycloak hostnames
- Verify SSL/TLS certificate validity
- Ensure proper firewall rules are in place

### Diagnostic Commands

#### Test JWKS URI Accessibility

```bash
# Test JWKS URI from MarkLogic server
curl -v "https://oauth.warnesnet.com:8443/realms/progress-marklogic/protocol/openid-connect/certs"
```

#### Decode JWT Token for Analysis

```bash
# Decode JWT token to inspect claims (requires jwt-cli or online decoder)
echo "$ACCESS_TOKEN" | jwt decode -

#Note: Requires the jwt-cli tools package

# Or using online tool: https://jwt.io/
```

#### Test Token Validity

```bash
# Validate token against Keycloak userinfo endpoint
curl -H "Authorization: Bearer $ACCESS_TOKEN" \
"https://oauth.warnesnet.com:8443/realms/progress-marklogic/protocol/openid-connect/userinfo"
```

### Best Practices for Troubleshooting

1. **Enable Logging Early**: Turn on debug logging before testing
2. **Test Components Separately**: Verify Keycloak token generation before testing MarkLogic integration
3. **Use Network Tools**: Employ tools like curl, nslookup, and telnet for connectivity testing
4. **Monitor Both Systems**: Watch logs on both Keycloak and MarkLogic simultaneously
5. **Document Configurations**: Keep detailed records of all configuration settings
6. **Test with Simple Scenarios**: Start with basic authentication before adding complex role mappings


