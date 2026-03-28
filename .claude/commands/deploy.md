Deploy the treasure game to GitHub Pages.

Run these steps in order:
1. Run `npm run build` to verify the build succeeds locally
2. Run `git status` to show pending changes
3. Run `git add .` to stage all changes
4. Inspect the staged diff, auto-generate a short descriptive commit message, and present it to me as a suggestion. Ask: "Use this message, or enter your own?" — if I approve, use it; if I decline, ask me to provide a custom message.
5. Commit with the chosen message and push to the main branch
6. Inform me that GitHub Actions will now automatically build and deploy
7. Tell me the live URL: https://bjmqfg83.github.io/claude-treasure-game/
8. Suggest running `gh run watch` or checking https://github.com/bjmqfg83/claude-treasure-game/actions to monitor the deployment progress
