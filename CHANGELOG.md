# Changelog

## v1.0.0

- **Reusable workflow:** resolves submitted pull request reviews with Gemini CLI and pushes the fix to the reviewed branch as a GitHub App.
- **Manual runs:** `pr-number` and `review-id` inputs apply the workflow to an earlier review.
- **Sandbox:** file tools only for Gemini, no token on disk, and protected paths checked before pushing.
- **Configuration:** inputs for the model, commit types, extra instructions, etc.
