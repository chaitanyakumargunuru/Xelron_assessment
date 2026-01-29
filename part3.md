# Part 3: Prompt Preparation

## Selected PR: Archivematica #1493 - Support LDAP Authentication Config via Environment Variables

---

## 3.1 Repository Context (200-300 words)

Archivematica is an open-source digital preservation system designed for cultural heritage institutions such as museums, libraries, and archives. The system processes digital objects through standardized workflows to create Archival Information Packages that can be preserved for decades or centuries. Archivematica implements the OAIS reference model, an ISO standard for long-term digital preservation.

The intended users are archivists, digital preservation specialists, and IT administrators working in memory institutions. These users need to ingest digital content (documents, images, audio, video, datasets) and create preservation packages that include original files, preservation copies, metadata, and documentation. The system handles format identification, validation, normalization, metadata extraction, and packaging according to standards like METS and PREMIS.

The problem domain Archivematica addresses is the challenge of ensuring long-term access to digital materials. Unlike physical objects, digital files require active management to remain accessible as file formats, software, and hardware evolve. Organizations need systems that document every action taken on digital objects, maintain fixity information to detect corruption, and store files in ways that maximize the chances of future accessibility.

Archivematica is built as a microservices architecture using Python and Django for the web Dashboard, with Gearman managing job queues between the MCPServer (orchestrator) and MCPClient (workers). The system integrates with storage backends, format policy registries, and external systems. Authentication is crucial because archivematica handles sensitive cultural materials and must control who can access, modify, or delete preservation packages. The system supports multiple authentication backends including local Django auth, CAS, Shibboleth, and LDAP for integration with institutional identity management systems.

---

## 3.2 Pull Request Description (200-300 words)

This pull request introduces the capability to configure LDAP authentication through environment variables instead of requiring modifications to Python settings files. Currently, users who want to enable LDAP authentication must edit the Dashboard's settings.py file to add LDAP server configuration, bind credentials, and search parameters. This approach creates several problems for modern deployments.

The specific changes introduce environment variable parsing in the Django settings file that reads LDAP configuration parameters from the system environment. The implementation adds environment variables like AUTH_LDAP_SERVER_URI for the LDAP server address, AUTH_LDAP_BIND_DN and AUTH_LDAP_BIND_PASSWORD for connection credentials, AUTH_LDAP_USER_SEARCH_BASE_DN and AUTH_LDAP_USER_SEARCH_BASE_FILTERSTR for user lookup configuration, and AUTH_LDAP_START_TLS for secure connection settings.

A new signals.py module was created to properly handle the LDAP user population signal from django-auth-ldap. This separates authentication concerns from the main settings file and provides a clean integration point. The implementation includes proper handling of import errors so the code works in contexts where LDAP libraries are not installed (such as the MCPClient workers).

The previous behavior required administrators to edit Python source files, rebuild containers, or create complex configuration file mounting schemes. This was error-prone, made version control difficult, and required understanding Django's settings structure. Credentials stored in source files also created security risks. The new behavior allows administrators to simply set environment variables in their docker-compose.yml, Kubernetes manifests, or systemd service files. The LDAP configuration becomes declarative and can be managed through standard secrets management tools. This aligns with twelve-factor app methodology and modern DevOps practices, making Archivematica deployments more maintainable and secure.

---

## 3.3 Acceptance Criteria (Minimum 5 criteria)

✓ When AUTH_LDAP_SERVER_URI is set as an environment variable, the system should connect to the specified LDAP server URL for authentication attempts.

✓ When AUTH_LDAP_BIND_DN and AUTH_LDAP_BIND_PASSWORD are provided, the system should successfully bind to the LDAP server using these credentials before attempting user searches.

✓ When AUTH_LDAP_USER_SEARCH_BASE_DN and AUTH_LDAP_USER_SEARCH_BASE_FILTERSTR are configured, the implementation should search for users in the specified base DN using the provided filter string with proper %(user)s substitution.

✓ The implementation should handle AUTH_LDAP_START_TLS environment variable by properly converting the string value "true"/"false" to boolean and enabling/disabling TLS accordingly.

✓ When no LDAP environment variables are set, the system should continue to function normally with existing authentication methods without errors or warnings.

✓ The implementation should maintain backward compatibility with existing file-based LDAP configurations while allowing environment variables to take precedence when set.

✓ When the django-auth-ldap library is not installed, the Dashboard should import without raising ImportError and gracefully skip LDAP configuration.

✓ The LDAP user population signal should correctly populate user attributes (username, email, groups) from LDAP directory into the Django user model.

---

## 3.4 Edge Cases (Minimum 3 cases)

**Edge Case 1: Missing Required Environment Variables**
If AUTH_LDAP_SERVER_URI is set but AUTH_LDAP_USER_SEARCH_BASE_DN is missing, the LDAP authentication will be partially configured. The implementation should either fail gracefully with a clear error message during startup or use sensible defaults. The system should not silently fail authentication attempts with unclear error messages.

**Edge Case 2: LDAP Server Unavailability During Startup**
When the Dashboard starts with LDAP configured but the LDAP server specified in AUTH_LDAP_SERVER_URI is not reachable, the application should still start successfully. Authentication attempts should fail with appropriate user-facing error messages rather than causing the entire application to crash. The system should handle network timeouts and connection errors without blocking the startup process.

**Edge Case 3: Boolean String Conversion**
The AUTH_LDAP_START_TLS variable comes from environment as a string but needs to be a Python boolean. The implementation must correctly handle various string representations including "true", "True", "TRUE", "false", "False", "FALSE", "1", "0", "yes", "no" and provide appropriate defaults for empty strings or malformed values. Invalid values should be logged and default to the safest option (TLS enabled).

**Edge Case 4: Special Characters in LDAP Credentials**
When AUTH_LDAP_BIND_PASSWORD contains special characters like quotes, backslashes, or environment variable syntax ($, %, etc.), the environment variable parsing must correctly preserve these characters without interpretation or escaping issues. The bind password should be used exactly as provided to authenticate with the LDAP server.

**Edge Case 5: MCPClient Context Import**
The MCPClient workers share some code paths with the Dashboard but don't need LDAP authentication since they don't handle user requests. When the settings file with LDAP configuration is imported in MCPClient context, the django-auth-ldap library may not be present. The implementation must catch the ImportError and continue execution without attempting to configure LDAP, while still allowing the MCPClient to function normally.

---

## 3.5 Initial Prompt (300-500 words)

**Task: Implement LDAP Authentication Configuration via Environment Variables**

You are working on Archivematica, a digital preservation system built with Django. Your task is to modify the Dashboard component to support LDAP authentication configuration through environment variables instead of requiring hard-coded settings in Python files.

**Context:**

Archivematica's Dashboard is a Django application that currently supports LDAP authentication through the django-auth-ldap library, but the configuration must be written directly in the settings.py file. This creates problems for containerized deployments where configuration should be managed through environment variables. Your implementation should allow administrators to configure LDAP authentication by setting environment variables in their deployment environment.

**Requirements:**

1. Modify `src/dashboard/src/main/settings.py` to read LDAP configuration from environment variables:
   - AUTH_LDAP_SERVER_URI: The LDAP server URL (e.g., "ldap://ldap.example.com")
   - AUTH_LDAP_BIND_DN: The DN to use when binding to LDAP (e.g., "cn=admin,dc=example,dc=org")
   - AUTH_LDAP_BIND_PASSWORD: The password for the bind DN
   - AUTH_LDAP_USER_SEARCH_BASE_DN: The base DN for user searches (e.g., "ou=users,dc=example,dc=org")
   - AUTH_LDAP_USER_SEARCH_BASE_FILTERSTR: The filter string for user searches (e.g., "(uid=%(user)s)")
   - AUTH_LDAP_START_TLS: Boolean flag to enable STARTTLS (string "true" or "false")

2. Create a new file `src/dashboard/src/main/signals.py` that:
   - Imports and handles the ldap_populate_user signal from django_auth_ldap.backend
   - Provides a signal handler for proper LDAP user population
   - Handles ImportError gracefully if django-auth-ldap is not installed

3. Update settings.py to:
   - Import the signals module with error handling
   - Parse environment variables with appropriate type conversions (especially AUTH_LDAP_START_TLS to boolean)
   - Configure AUTH_LDAP_* Django settings only when environment variables are present
   - Maintain backward compatibility with existing configurations

4. Update documentation in `src/dashboard/install/README.md` to include a table describing each environment variable, its purpose, and example values.

**Testing Requirements:**

Ensure your implementation can be tested with an OpenLDAP Docker container using these settings:
```
LDAP_ORGANISATION: "Example Org"
LDAP_DOMAIN: "example.org"
LDAP_ADMIN_PASSWORD: "admin"
```

**Acceptance Criteria Reference:**

Your implementation must satisfy all acceptance criteria listed in section 3.3, particularly:
- Proper environment variable parsing and type conversion
- Graceful handling of missing django-auth-ldap library
- Backward compatibility with existing authentication configurations
- Clear documentation of configuration options

**Edge Cases to Consider:**

Pay special attention to the edge cases identified in section 3.4:
- Partial LDAP configuration (some variables set, others missing)
- LDAP server unavailability during application startup
- Various string representations of boolean values
- Special characters in passwords
- Import errors in MCPClient context

**Code Quality:**

Follow Django and Python best practices:
- Use `os.environ.get()` with appropriate defaults
- Include proper error handling and logging
- Keep the signals module focused and single-purpose
- Add clear comments explaining the configuration logic
- Ensure PEP 8 compliance

**Deliverables:**

1. Modified settings.py with environment variable configuration
2. New signals.py module
3. Updated README.md documentation
4. Confirmation that the implementation works with the test LDAP container configuration

---

## Integrity Declaration

I declare that all written content in this assessment is my own work, created without the use of AI language models or automated writing tools. All technical analysis and documentation reflects my personal understanding and has been written in my own words.
