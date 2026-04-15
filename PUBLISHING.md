# Publishing checklist — @singing-duck/capture-duck

## Before you publish to npm

1. **Update package metadata**
   - Set the final npm package name in `package.json` (scope or unscoped).
   - Set `repository.url` to your new repo.
   - Set `author` if needed.

2. **Run tests**

   ```bash
   npm test
   ```

3. **Dry-run package contents**

   ```bash
   npm pack --dry-run
   ```

   Confirm only expected files are included (`src`, `README.md`, `PUBLISHING.md`).

4. **Login to npm**

   ```bash
   npm login
   ```

5. **Publish**

   ```bash
   npm publish --access public
   ```

   Use `--access public` for scoped public packages.

## Suggested first release

- Start with `0.1.0`.
- After publishing, tag your git repo with `v0.1.0`.

## Post-publish smoke test

In a fresh folder:

```bash
npm init -y
npm install @singing-duck/capture-duck
node --input-type=module -e "import { parseStackTrace } from '@singing-duck/capture-duck'; console.log(parseStackTrace(new Error('x').stack).length)"
```
