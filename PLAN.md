# Calorie Tracker v1: Spec Summary

Output of the planning interview. Status: spec agreed, implementation plan not yet written.

## Product

An Android-only, Cal AI-style calorie tracker. The user photographs a meal, and an OpenAI vision model returns calories, protein, carbs and fat. Daily targets are personalized to each user.

## Users

Individuals tracking weight goals (lose / maintain / gain). Metric units only (kg, cm).

## Scope

### In scope for v1

- Google sign-in through Clerk, with the auth layer designed so Apple can be added later
- Onboarding: age, sex, height (cm), weight (kg), activity level, goal
- Daily calorie and macro targets via Mifflin-St Jeor, with progress against them
- Camera-only meal capture and AI analysis
- Edit a meal's totals and delete a meal
- Home, History (grouped by day), and Profile screens
- In-app account deletion
- A daily scan cap per user

### Out of scope for v1

Manual food entry, barcode scanning, gallery uploads, weight charts, reminders and notifications, water and exercise tracking, social features, billing, Apple sign-in, iOS, ingredient-level breakdowns, weekly adaptive targets.

## Core flows

1. **First launch:** Google sign-in, then onboarding, then targets are calculated and stored, then Home.
2. **Scan:** camera capture, upload straight to ImageKit with a signed token from our API, API creates a pending meal and triggers the Trigger.dev task, app subscribes via Realtime, and the result appears on Home. If the task fails after retries, the user sees "couldn't analyze" with a retry button.
3. **Review:** Home shows today's totals vs target plus today's meals. History shows past days. Users can edit totals or delete a meal.
4. **Profile:** editing info recalculates targets from that day forward, and past days keep the target they had. Users can also sign out or delete their account.

## Data model (draft)

- **users:** Clerk ID, age, sex, height\_cm, weight\_kg, activity\_level, goal
- **daily\_targets:** user, effective date, calories, protein, carbs, fat (a snapshot per change, so history stays correct)
- **meals:** user, ImageKit file ID and URL, name, calories, protein, carbs, fat, status (pending / complete / failed), `logged_at` in UTC, `local_date`, edited flag
- The daily scan count is derived from meals per `local_date`.

## Architecture

- **App:** Expo and React Native on Android, with Sentry.
- **API:** Expo Router API routes on EAS Hosting. They verify Clerk tokens, issue ImageKit upload tokens, enforce the scan cap server-side, create pending meals, and trigger tasks.
- **Background work:** a Trigger.dev task reads the ImageKit image, calls the OpenAI vision model for structured output, writes to Neon, and retries on failure. The app follows progress via Realtime.
- **Data:** Neon Postgres, with dev and prod branches.
- **Monitoring:** Sentry covers the app, API routes, and Trigger.dev task failures.
- **Testing:** unit tests for target math and the Trigger.dev task, with manual QA for the UI.
- **Environments:** dev and prod only.

## Assumptions

Items where no firm answer was given.

1. Daily scan cap is **10 per day**, adjustable. A scan that ultimately fails doesn't count against it.
2. v1 is **free**, with no billing or paywall.
3. All optional extras left unselected are **out of scope**.
4. Meals store UTC time plus the user's local date at logging, so travel doesn't rewrite history.
5. Activity multipliers and goal adjustments (roughly ±500 kcal/day) use standard defaults, with a macro split chosen at plan time.
6. A photo that isn't food is treated as a failed analysis.
7. The app requires a network connection, with no offline logging.
8. English only. The app is tutorial-grade, not accessibility-audited.
9. The ORM and query layer (e.g. Drizzle) are undecided, to be proposed in the plan.
10. The exact OpenAI model, its pricing, and all library versions (Expo SDK, Clerk, Trigger.dev, Sentry) will be verified against current docs before the plan is final.

## Open risks

- **EAS Hosting:** newer platform, so limits, cold starts, and Sentry server-side support need checking.
- **Clerk Google sign-in on Android:** likely needs a development build rather than Expo Go.
- **Trigger.dev Realtime in Expo:** needs scoped public access tokens, and the client needs confirming in React Native.
- **Neon from serverless routes:** connection pooling and the serverless driver choice.
- **Health safety:** aggressive goals can produce unsafe calorie targets. Add a calorie floor and a "not medical advice" note.
- **AI accuracy and trust:** estimates are approximate, and editing totals is the only recourse.
- **Google Play:** needs a privacy policy and data-safety form, since we collect health-adjacent personal data.
- **Upload security:** signed token expiry and file size limits for ImageKit.
- **Cost:** OpenAI spend depends on the cap and on whether images are resized before upload.

## Next step

Turn this spec into an implementation plan (plan.md).
