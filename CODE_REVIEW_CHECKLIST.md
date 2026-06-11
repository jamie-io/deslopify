# Code Review Checklist

This checklist provides a comprehensive guide for reviewers to ensure code quality and consistency across the Deslopify project.

## 1. Code Quality

- [ ] **Code Formatting**: Code follows the project's style guidelines
- [ ] **Variable Naming**: All variables have descriptive names
- [ ] **Comments**: Functions and complex logic have clear comments
- [ ] **TypeScript/JavaScript**: No syntax errors, proper ES6+ syntax
- [ ] **Error Handling**: Proper error handling and edge cases covered
- [ ] **Security**: No sensitive data (API keys, tokens) exposed
- [ ] **Performance**: No performance anti-patterns identified

## 2. Tests

- [ ] **Unit Tests**: New functions have adequate unit test coverage
- [ ] **Integration Tests**: New features tested in integration context
- [ ] **Edge Cases**: Error conditions and edge cases are tested
- [ ] **Existing Tests**: All existing tests continue to pass
- [ ] **Test Coverage**: No regression in test coverage

## 3. Documentation

- [ ] **README**: Updated for new features or breaking changes
- [ ] **Function Documentation**: Added JSDoc comments for new functions
- [ ] **API Documentation**: Parameters and return values documented
- [ ] **Code Comments**: Complex logic explained
- [ ] **Examples**: Code examples included where helpful

## 4. Manifest and Configuration

- [ ] **manifest.json**: Updated if permissions or features changed
- [ ] **manifest_version**: Updated if MV3 → MV2 or vice versa
- [ ] **Host Permissions**: No unnecessary permissions added
- [ ] **Web Accessibile Resources**: Properly configured for new scripts

## 5. Architecture and Design

- [ ] **Modularity**: Code follows the modular architecture
- [ ] **Separation of Concerns**: Clear separation between content, lib, and UI
- [ ] **Dependencies**: No unnecessary dependencies introduced
- [ ] **State Management**: Proper state management patterns used
- [ ] **Error Recovery**: Graceful degradation on errors

## 6. Chrome Extension Specific

- [ ] **Content Scripts**: Properly configured with correct run_at and all_frames
- [ ] **Storage**: Efficient use of chrome.storage.sync
- [ ] **Permissions**: Minimal required permissions
- [ ] **API Usage**: YouTube API used within policy
- [ ] **Compatibility**: Cross-browser compatibility considered

## 7. Code Review Specific Points

### Functions and Methods
- [ ] Function has clear purpose and responsibility
- [ ] Function parameters are properly validated
- [ ] Function has comprehensive JSDoc documentation
- [ ] Function is unit testable
- [ ] Function follows existing patterns

### Variables and Constants
- [ ] Variables have descriptive names
- [ ] Constants are properly named (UPPER_CASE)
- [ ] Variables are scoped appropriately
- [ ] Magic numbers/strings replaced with constants
- [ ] No duplicate variables

### Imports and Requires
- [ ] Imports are ordered alphabetically
- [ ] Duplicate imports removed
- [ ] Unused imports removed
- [ ] Type imports are properly distinguished

### Error Handling
- [ ] Errors are caught and handled gracefully
- [ ] User-friendly error messages provided
- [ ] Logging is appropriate (not exposing sensitive data)
- [ ] API failures are properly recovered from

## 8. Performance

- [ ] Cache usage is optimal
- [ ] Debouncing is used for frequent operations
- [ ] DOM manipulation is minimized
- [ ] API calls are properly throttled
- [ ] Memory leaks are prevented

## 9. Security Considerations

- [ ] No API keys or secrets hardcoded
- [ ] User data is properly encrypted in storage
- [ ] CORS policies are respected
- [ ] No XSS vulnerabilities in generated HTML
- [ ] Proper content security policies if needed

## 10. Compatibility

- [ ] Tested on Chrome and Firefox (if applicable)
- [ ] Handles mobile YouTube (m.youtube.com)
- [ ] Works with YouTube's API changes
- [ ] Graceful fallback for unsupported features
- [ ] No breaking changes to existing functionality

## 11. Maintainability

- [ ] Code is easy to understand and modify
- [ ] Dependencies are properly managed
- [ ] Code is well-structured and organized
- [ ] Future extensions are considered
- [ ] Technical debt is minimized

## 12. Project-Specific Checklist

### Core Features (Title, Thumbnail, Description, Audio, Channel)
- [ ] Each feature has its own dedicated test suite
- [ ] Feature functions are properly isolated
- [ ] Feature settings are properly saved and restored
- [ ] Feature can be disabled individually

### Cache System
- [ ] Cache entries are properly serialized/deserialized
- [ ] Cache eviction works correctly
- [ ] Cache memory usage is bounded
- [ ] Cache respects storage limits

### API Client
- [ ] API requests have proper error handling
- [ ] API authentication is properly managed
- [ ] API responses are properly validated
- [ ] Rate limiting is respected

### DOM Manipulation
- [ ] DOM operations are idempotent
- [ ] DOM changes are minimal and efficient
- [ ] Event listeners are properly cleaned up
- [ ] MutationObservers are correctly configured

## 13. Testing Checklist

### Unit Tests
- [ ] Test covers all public API functions
- [ ] Test includes edge cases and error conditions
- [ ] Test uses appropriate mocking
- [ ] Test follows testing framework conventions
- [ ] Test names clearly describe what is being tested

### Integration Tests
- [ ] Test real-world scenarios
- [ ] Test with actual YouTube pages
- [ ] Test extension loading and initialization
- [ ] Test feature interactions

## 14. Final Review

- [ ] All requirements are met
- [ ] Code passes all tests
- [ ] Code passes linting
- [ ] Documentation is complete
- [ ] Tests are comprehensive
- [ ] Code follows project conventions
- [ ] No regressions introduced
- [ ] Ready for production

## Code Review Process

1. **Initial Review**: Quick scan for obvious issues
2. **Technical Review**: Deep dive into implementation details
3. **Documentation Review**: Check documentation completeness
4. **Testing Review**: Verify test coverage and quality
5. **Final Sign-off**: Ensure all requirements are met

## When to Request Additional Review

- Changes affect core functionality
- Changes introduce new public APIs
- Changes affect performance characteristics
- Changes affect security considerations
- Changes affect compatibility
- Significant refactoring
- Major architecture changes

---

This checklist is a living document. Feel free to update it based on lessons learned from the review process.
