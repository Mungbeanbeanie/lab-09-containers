# Lab 09: Knowledge check

**Answer each question below it, in your own words, in one to three sentences.** Replace the `...` below **Your answer:** with your text.

## 1. Credentials in the image?

Assume you want to share your image with others and make it publicly available on `ghcr.io`, so that anyone can pull it and run the analysis. A teammate suggests adding `COPY .aws /home/app/.aws` to the `Dockerfile`, so that nobody has to type `-v ~/.aws:/home/app/.aws:ro` any more. (This is a thought experiment: keep your own package private.)

**Your answer:** Give two reasons why this is a bad idea. Think of who could then read your credentials, and of what happens when your Learner Lab session ends.

First, anyone who can pull a public image can read every file a `COPY` put into it, so my AWS keys and session token would be readable by anyone. They could use my Learner Lab account and reach my buckets and my SSM parameter, which holds the Atlas password. Second, Learner Lab credentials are temporary and expire when the session ends, so the copied keys would soon stop working (`ExpiredToken`), and every new session would mean rebuilding and re-pushing the image. Mounting `~/.aws` read-only at run time keeps the credentials out of the image and always gives the container my current ones.

## 2. Where did the Atlas password come from?

In Step 4, you ran the analysis once with `-v ~/.aws:/home/app/.aws:ro` only, without the three `-e` options. The container still wrote to your Atlas cluster.

*Hint:* Read `get_atlas_settings()` in `src/book_analysis/analyze.py`.

**Your answer:** Explain, step by step, where the container found the Atlas settings in that run.

Without the `-e` options, `MONGODB_ATLAS_URL` is not set in the container, so `get_atlas_settings()` skips the environment-variable branch. It then calls `boto3`'s SSM client, which authenticates with the Learner Lab credentials mounted at `/home/app/.aws`, and reads the `mongodb_credentials` parameter from Lab 07 with `WithDecryption=True`. That parameter's JSON value supplies the `url`, `user`, and `password` that `connect()` passes to `MongoClient`.

## 3. Same tag, different image

Run `docker images book-analysis`, and note the image ID of `book-analysis:0.1.0`. Then change the `description` in the `LABEL` of your `Dockerfile`, build again with the same command, `docker build -t book-analysis:0.1.0 .`, and run `docker images book-analysis` again.

**Your answer:** Paste the image ID from before and after the rebuild. Why is it a problem for someone who uses your image that `0.1.0` now names a different image, and what should you have done instead?

Before: `a6cf74d17a30`. After: `202de42a9cb2`. Someone who pulled or cited `0.1.0` expects that tag to mean one exact image, but it now silently points to a different one, so their results are no longer reproducible and they can't tell what changed. I should have raised the `version` in `pyproject.toml` and built under a new tag, such as `0.1.1`, leaving `0.1.0` unchanged (or referred to images by digest).
