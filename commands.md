# Standing Commands

These rules are **always** active. No exceptions.

1. **Minimum tokens, maximum quality.** Be concise. No fluff. Tokens are limited, work is not.
2. **Always save session chats.** At the end of every session or conversation, store what we discussed and what's left in the respective `projects/<project>/modules/<module>/` folder. This is **high priority**.
3. **Update manish-profile.md** whenever something new is learned about Manish.
4. **Never re-explain from scratch.** Always read existing project/module context before starting work.
5. **Separate logic into separate functions.** Match Manish's coding style.
6. **Prefer JS/Node unless told otherwise.** Golang only for utilities-go. TypeScript for CRM-Backend migration.
7. **Full Desktop access granted.** Manish has given permission to perform **any operation** across `~/Desktop/` and subdirectories without asking — this includes all read operations (`ls`, `head`, `cat`, `grep`, `find`, `glob`, file reads, directory listings, git log/status/diff, etc.) and all non-write commands (running servers, tests, installs, bash commands, subagent searches, etc.). The only thing that requires confirmation is **writing/editing files outside the current project**. Never prompt for permission on read-only or non-destructive actions.
8. **Frontend integration: Do NOT change the UI unless absolutely necessary.** When wiring backend data to frontend, only change the data layer — keep existing UI/UX intact. Ask Manish before any UI modifications.
9. **Testing permission granted.** During testing (curl, API calls, server restarts, database queries, running dev servers, killing processes, etc.), execute all commands without asking for permission. This includes destructive test operations like deleting test data, restarting servers, and running any bash/curl commands needed to verify functionality.
