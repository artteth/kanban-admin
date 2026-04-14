# iOS 12 Compatibility - WORK IN PROGRESS

## Current Status

✅ **CSS Improvements Added:**
- Added `-webkit-text-size-adjust: 100%` for better iOS Safari support
- Added `-webkit-backdrop-filter` prefixes for modal overlays
- Improved text rendering on older iOS devices

⚠️ **JavaScript ES5 Conversion - PENDING:**
The JavaScript code in these HTML files contains ES6+ syntax (arrow functions, const/let, template literals) that is not compatible with iOS 12 Safari. 

## What Needs to Be Done

### Option 1: Manual Conversion (Recommended for accuracy)
Convert all ES6+ syntax to ES5-compatible syntax:

1. **Replace `const` and `let` with `var`**
   ```javascript
   // BEFORE:
   const API_URL = '...';
   let tasks = [];
   
   // AFTER:
   var API_URL = '...';
   var tasks = [];
   ```

2. **Convert arrow functions to regular functions**
   ```javascript
   // BEFORE:
   tasks.filter(t => t.status === 'todo')
   
   // AFTER:
   tasks.filter(function(t) { return t.status === 'todo'; })
   ```

3. **Convert template literals to string concatenation**
   ```javascript
   // BEFORE:
   `Hello ${name}, you have ${count} tasks`
   
   // AFTER:
   'Hello ' + name + ', you have ' + count + ' tasks'
   ```

4. **Remove optional chaining `?.` and nullish coalescing `??`**
   ```javascript
   // BEFORE:
   const value = obj?.prop ?? 'default';
   
   // AFTER:
   var value = (obj && obj.prop) !== undefined && (obj && obj.prop) !== null ? (obj && obj.prop) : 'default';
   ```

### Option 2: Use Babel (Semi-automated)
1. Extract all `<script>` content from HTML files into separate `.js` files
2. Run Babel transpilation:
   ```bash
   npm install --save-dev @babel/core @babel/cli @babel/preset-env
   npx babel app.js --out-file app.es5.js
   ```
3. Put the transpiled code back into HTML files

### Files Requiring Conversion:
- `index.html` - Main Kanban board (contains ~1000 lines of JS)
- `admin.html` - Admin panel (contains ~1700 lines of JS)  
- `select_user.html` - User selection page (contains ~500 lines of JS)

## Testing on iOS 12

After conversion, test on actual iOS 12 device or BrowserStack:
1. Open the app in Safari
2. Check console for syntax errors
3. Verify all buttons and interactions work
4. Test task creation, movement, and deletion

## Resources
- [Can I Use ES6](https://caniuse.com/#feat=es6)
- [iOS 12 Safari Compatibility](https://caniuse.com/#feat=safari12-1)
- Full guide: See `IOS_12_COMPATIBILITY_GUIDE.md`

## Next Steps
Complete the ES5 conversion of all JavaScript code in the HTML files to ensure full iOS 12 compatibility.
