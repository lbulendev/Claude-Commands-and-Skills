# Document Feature Command
Analyze the feature or code changes provided in $ARGUMENTS.

### Detection Step
Determine if the scope is:
- **Documentation**: Documentation
- **Frontend**: Primarily UI components, state management, or styling.
- **Backend**: API routes, database schemas, or server-side logic.
- **Full-stack**: Significant changes spanning both UI and server.

### Documentation Requirements
Adjust the output documentation based on the detected scope:

1. **If Documentation**:
   - Focus on Documentation

2. **If Frontend**:
   - Focus on Component Props, UX/UI behavior, and Accessibility.
   - Include a section for "Visual Verification" (Screenshots/DevTools).

3. **If Backend**:
   - Focus on API Endpoints (Method, Path), Request/Response schemas, and Error Handling.
   - Include a section for "Server Logs/Database Impact".

4. **If Full-stack**:
   - Document the end-to-end data flow from UI to Database.
   - Combine both Frontend and Backend requirements above.

### Output Format
1. **If Documentation**:
  `docs/doc/{filename}.md`

2. **Else**:
  `docs/dev/{filename}.md`
   - Navigate to the specified URL using the browser, capture the viewport, and automatically generate and insert the markdown syntax with an appropriate caption into the documentation.

