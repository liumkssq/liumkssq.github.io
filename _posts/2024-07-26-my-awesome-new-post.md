    ---
    layout: post                     # Specifies the layout file (_layouts/post.html)
    title: "My Awesome New Post"     # The title of your post
    subtitle: "A quick look into something cool" # The subtitle displayed below the title
    date: 2024-07-26 10:30:00       # The date and time of publication (use YYYY-MM-DD format at minimum)
    author: "Your Name"              # Your name
    header-img: "img/post-bg-universe.jpg" # Optional: Path to an image for the header background
    catalog: true                    # Optional: Set to true to generate a table of contents for the post
    tags:                            # Optional: Add relevant tags
      - Tech
      - Blogging
      - Example
    ---

    # Start writing your blog content here in Markdown...

    This is the first paragraph of your post. You can use standard Markdown syntax.

    ## Subheadings work too

    *   And lists
    *   Are easy

    You can include code blocks:

    ```javascript
    console.log("Hello, Jekyll!");
    ```
    ```
4.  **Write your content** below the second `---` line using Markdown.

**After Creating the Post (Either Method):**

1.  **Save the file.**
2.  **(Optional) Preview locally:** If you have Jekyll set up locally, run `bundle exec jekyll serve` and check `http://localhost:4000` to see how it looks.
3.  **Commit and push the changes** to your GitHub repository:
    ```bash
    git add _posts/YYYY-MM-DD-your-title.md
    # Or git add . to add all changes
    git commit -m "Add new blog post: My Awesome New Post"
    git push origin master
    ```

GitHub Pages will automatically rebuild your site, and your new post should appear shortly.