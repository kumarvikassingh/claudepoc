You need to check total file count and total lines of code without comment lines. 

Do the following:
1. Count total file count in the repository. Only count the files which are source code files (e.g., .js, .ts, .tsx, .jsx, .css, .html, etc.). Do not count configuration files (e.g., .env, .gitignore, package.json, etc.) or documentation files (e.g., README.md).
2. Count total lines of code in the repository, excluding comment lines and blank lines.
To count the total file count and total lines of code in the repository, you can use the following commands in your terminal:
1. To count total file count:
```bashfind . -type f \( -name "*.js" -o -name "*.ts" -o -name "*.tsx" -o -name "*.jsx" -o -name "*.css" -o -name "*.html" \) | wc -l
``` 2. To count total lines of code, excluding comment lines and blank lines:
```bashfind . -type f \( -name "*.js" -o -name "*.ts" -o -name "*.tsx" -o -name "*.jsx" -o -name "*.css" -o -name "*.html" \) -exec grep -vE '^\s*(//|/\*|\*|$)' {} + | wc -l
``` Make sure to run these commands from the root directory of your repository. The first command will give you the total file count, and the second command will provide you with the total lines of code, excluding comments and blank lines.