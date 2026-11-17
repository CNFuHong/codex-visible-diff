# Examples

Each block below is the ENTIRE exec command. Nothing follows the closing delimiter. Run any verification as a separate command.

## Add a file

```text
apply_patch <<'EOF'
*** Begin Patch
*** Add File: src/hello.txt
+hello
+world
*** End Patch
EOF
```

## Update a file (context lines, no line numbers)

```text
apply_patch <<'EOF'
*** Begin Patch
*** Update File: src/app.go
@@
 func main() {
-	println("old")
+	println("new")
 }
*** End Patch
EOF
```

## Delete a file

```text
apply_patch <<'EOF'
*** Begin Patch
*** Delete File: src/old.txt
*** End Patch
EOF
```

## Multiple files in one patch

```text
apply_patch <<'EOF'
*** Begin Patch
*** Update File: a.go
@@
-old
+new
*** Add File: b.txt
+beta
*** End Patch
EOF
```

## Windows / PowerShell notes

- The same bash-style heredoc is intercepted under PowerShell (the parser uses bash extraction for PowerShell/Cmd too). Do NOT switch to PowerShell here-strings (`@' … '@`); those are not the intercepted form.
- Keep the command pure: no `Write-Output`, `Get-Content`, `;`, or `&&` after the closing delimiter.
- If the patch body contains a standalone line equal to your delimiter, choose another unique delimiter (e.g. `PATCH`).
- Paths are relative to the exec working directory; set the working directory to the repo root you intend to edit.

## Anti-patterns (write the file, but emit NO diff)

```text
apply_patch $p                            # variable, not a literal heredoc
apply_patch "*** Begin Patch …"           # quoted argument, not a heredoc
apply_patch <<'EOF' … EOF; Get-Content …  # trailing commands break extraction
codex.exe --codex-run-as-apply-patch $p   # subprocess bypasses interception
python -c "open(...).write(...)"          # shell rewrite
sed -i … / perl -pi … / echo > / cat > / tee
```
