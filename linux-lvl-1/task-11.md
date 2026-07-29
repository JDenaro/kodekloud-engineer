# Task 11: String Replacement

At xFusionCorp Industries, the Stratos Datacenter houses a jump host server that stores template XML files essential for the Nautilus application. Prior to their use, these files need to be populated with valid data. As part of regular maintenance, the system administration team utilizes various string and file manipulation commands to prepare these templates.

Your task is to substitute all occurrences of the string `Text` with `Echo-Location` within the XML file located at `/root/nautilus.xml` on the jump host server.

## Task Requirements

1. Work directly on the Jump Host.
2. Modify `/root/nautilus.xml`.
3. Replace every occurrence of the exact string `Text` with `Echo-Location`.

## Solution

The replacement can be performed in place with a single `sed` substitution command. The global substitution flag is necessary because the same string can appear more than once on a line.

The search is case-sensitive. It replaces the uppercase value `Text` while leaving lowercase XML element names such as `<text>` unchanged.

### 🔄 Step 1: Replace every occurrence

Run the command directly from `thor@jump-host`:

```bash
sudo sed -i 's/Text/Echo-Location/g' /root/nautilus.xml
```

The command returned to the prompt without an error:

```text
thor@jump-host ~$ sudo sed -i 's/Text/Echo-Location/g' /root/nautilus.xml
thor@jump-host ~$
```

> **Why:** `sudo` provides the privileges required to modify a file under `/root`. `sed` is a stream editor used to transform text. The `-i` option edits the specified file in place instead of writing the transformed content only to standard output. The substitution expression uses the form `s/search/replacement/g`: `Text` is the exact string to find, `Echo-Location` is its replacement, and the trailing `g` applies the substitution to every matching occurrence on each line. `/root/nautilus.xml` is the target file.

### ✅ Step 2: Verify the replacement

Display the lines containing the new value:

```bash
sudo grep -n 'Echo-Location' /root/nautilus.xml
```

The command produced many matches. This abbreviated excerpt shows the beginning and end of the actual output:

```text
11:      <text>Echo-Location</text>
23:      <text>Echo-Location</text>
35:      <text>Echo-Location</text>
47:      <text>Echo-Location</text>
59:      <text>Echo-Location</text>
71:      <text>Echo-Location</text>
...
793:      <text>Echo-Location</text>
805:      <text>Echo-Location</text>
817:      <text>Echo-Location</text>
829:      <text>Echo-Location</text>
```

> **Why:** `grep` prints lines that match the supplied pattern. The `-n` option prefixes each matching line with its line number, making it easier to see that `Echo-Location` appears throughout the XML file. The lowercase `<text>` tags remain intact because both `sed` and `grep` treat uppercase and lowercase characters as different by default.

Confirm that the original uppercase string no longer appears:

```bash
sudo grep -n 'Text' /root/nautilus.xml
```

The command returned no matching lines:

```text
thor@jump-host ~$ sudo grep -n 'Text' /root/nautilus.xml
thor@jump-host ~$
```

> **Why:** Searching for the original exact string provides the complementary verification. No output means that no uppercase `Text` occurrence remains in the file, while the previous command proves that the replacement value was written successfully.

## Best Practices

- **Use the global substitution flag.** The `g` flag ensures that every match on a line is replaced, not only the first.
- **Respect case sensitivity.** Searching for `Text` avoids modifying lowercase XML tags named `text`.
- **Edit the intended file directly.** `sed -i` applies the transformation to `/root/nautilus.xml` without creating an unrelated output file.
- **Verify both sides of the transformation.** Confirm that the new value exists and that the old value no longer appears.
- **Abbreviate repetitive verification output.** A representative excerpt keeps the guide readable while preserving evidence from the successful lab.

### 📚 Official Documentation

- [GNU sed manual](https://www.gnu.org/software/sed/manual/sed.html)
- [grep(1) Linux manual page](https://man7.org/linux/man-pages/man1/grep.1.html)
