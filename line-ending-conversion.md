## Outline
The idea is to start by making sure that all text file in the working tree have consistent CRLF line endings. This can be accomplished by exploiting git's line endings normalization functionality. When all files in the working have CRLF line endings, disable normalization and commit all files with CRLF line endings. This involves the following steps:

1) Enable line ending normalization on repository (update .gitattributes)
2) Commit all files with normalized line endings.
3) Ensure CRLF line endings on all files by doing a fresh checkout of all files.
4) Disable line ending normalization on repository (update .gitattributes)
5) Commit normalization configuration and all files with CRLF line endings
6) Squash the two commits

The files are first normalized because git will not do LF -> CRLF transformation
on files with mixed line endings on checkout. On the other hand, it will gladly
normalize files with mixed line endings.

## Implementation
Add line `* text=auto` at the top of .gitattributes (1)
```
## 2) Commit all files with normalized line endings.
# Stage files with normalized line ending to index
git add --renormalize .

# Commit staged changes
git commit -m "Enabled line ending normalization and renormalized line endings"

## 3) Ensure CRLF line endings on all files by doing a fresh checkout of all files.
# Delete all tracked files from working directory
git ls-files -z | xargs -0 rm -f

# Checkout all files from index - this will apply the LF -> CRLF transformation
git checkout-index -a

# Update index
git add -u
```

Update first line of .gitattributes to match `* -text` (4)
```
## 5) Commit normalization configuration and all files with CRLF line endings
# Stage files with CRLF line endings
git add --renormalize .

# Stage .gitattributes
git add .gitattributes

# Commit
git commit -m "Converted all line endings to CRLF"

## 6) Squash the two commits
git reset --soft HEAD~2
git commit -m "Converted all text files to CRLF line endings and disabled line ending normalization"
```
