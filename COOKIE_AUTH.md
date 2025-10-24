# Session Cookie Authentication

The Superset MCP now uses **session cookie authentication** instead of username/password login via the API.

## Why Cookie Authentication?

- More secure - no need to store credentials
- Works with any authentication method (LDAP, OAuth, SSO, etc.)
- Simpler implementation
- Uses your existing browser session

## How to Authenticate

### Step 1: Get Your Session Cookie

1. Open Superset in your browser and log in
2. Open Developer Tools (F12 or right-click → Inspect)
3. Go to the **Application** tab (Chrome) or **Storage** tab (Firefox)
4. Navigate to **Cookies** → Your Superset URL
5. Find the cookie named `session`
6. Copy the entire **Value** (it will be a long string)

### Step 2: Set the Cookie in MCP

Use the `superset_auth_set_session_cookie` tool:

```python
{
  "session_cookie": "your-very-long-session-cookie-value-here"
}
```

The MCP will:
- Validate the cookie by calling `/api/v1/me/`
- Store it in `.superset_session` for future use
- Automatically load it on restart
- Use it for all subsequent API requests

### Step 3: Start Using the MCP

Once authenticated, you can use all the other tools like:
- `superset_dashboard_list`
- `superset_chart_create`
- `superset_database_list`
- etc.

## Session Expiration

If your session expires (you'll get a 401 error), simply:
1. Get a fresh session cookie from your browser
2. Call `superset_auth_set_session_cookie` again with the new cookie

## Security Notes

- The session cookie is stored in `.superset_session` (added to .gitignore)
- Never commit this file to version control
- The cookie has the same expiration as your browser session
- Treat the cookie like a password - it grants full access to your Superset account

## Example Workflow

```bash
# 1. Start the MCP server
# 2. In Claude Code or your MCP client:

# Set session cookie (one time)
superset_auth_set_session_cookie(session_cookie="eyJ...")

# Now you can use other tools
superset_dashboard_list()
superset_chart_list()
superset_database_list()
```

## Troubleshooting

### "Invalid session cookie" Error
- Make sure you copied the entire cookie value
- Check that you're logged into Superset in your browser
- Try logging out and back in to get a fresh cookie

### "Session expired" Error
- Your browser session has expired
- Get a new cookie from your browser
- Set it again using `superset_auth_set_session_cookie`

### Cookie Not Persisting
- Check file permissions on `.superset_session`
- Make sure the file is being created in the correct directory
