```powershell
git checkout main
cat > .gitattributes << 'EOF'
.env merge=ours
.env.development merge=ours
.gitignore merge=ours
EOF

git add .gitattributes
git commit -m "Protect env files and gitignore from being overwritten during uat merges"
git push origin main
```

Remove
```powershell
git rm --cached file_name
```

Enable
```powershell
git config merge.ours.driver true
```
