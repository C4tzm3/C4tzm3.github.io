# Setting Up Comments and Discussions

Your blog now has comments support using Giscus (GitHub Discussions). Follow these steps to activate it:

## Step 1: Enable GitHub Discussions

1. Go to your repository: https://github.com/C4tzm3/C4tzm3.github.io
2. Click **Settings** tab
3. Scroll down to **Features** section
4. Check the box: ☑ **Discussions**
5. Click **Set up discussions**

## Step 2: Create Comments Category

1. Go to **Discussions** tab
2. Click **Categories** (pencil icon)
3. Create a new category:
   - **Name:** Comments
   - **Description:** Blog post comments
   - **Format:** Announcement (only you can create new discussions)

## Step 3: Get Giscus Configuration

1. Go to: https://giscus.app
2. Enter your repository: `C4tzm3/C4tzm3.github.io`
3. Scroll down to **Page ↔️ Discussions Mapping**
   - Select: `pathname`
4. Scroll down to **Discussion Category**
   - Select: `Comments` (the category you just created)
5. Copy the `data-repo-id` and `data-category-id` values from the generated script

## Step 4: Update Comments Configuration

Edit `_includes/comments.html` and replace:
- `REPO_ID_PLACEHOLDER` with your actual repo ID
- `CATEGORY_ID_PLACEHOLDER` with your actual category ID

Example:
```html
data-repo-id="R_kgDOabcdefg"
data-category-id="DIC_kwDOabcdefg4Cabcd"
```

## Step 5: Push and Test

```bash
git add .
git commit -m "Enable comments with Giscus"
git push origin main
```

Wait 2-3 minutes for GitHub Pages to rebuild, then visit any blog post. You should see a comments section at the bottom!

## How It Works

- **Comments on posts:** Visitors can comment on any blog post using their GitHub account
- **Questions/Requests:** Visitors can ask questions or request topics on the main page using GitHub Discussions
- **Privacy:** All comments are stored in your GitHub Discussions (not a third-party service)
- **Moderation:** You control all discussions and can moderate/delete comments

## Features Added

✅ Comments on every blog post
✅ Question/inquiry form on homepage (links to GitHub Discussions)
✅ Splunk posts organized under `/notes/splunk/`
✅ All privacy-friendly using GitHub's infrastructure

Need help? Check the Giscus documentation: https://giscus.app
