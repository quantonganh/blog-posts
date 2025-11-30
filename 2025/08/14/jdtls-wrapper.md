---
title: "Helix returns \"No definition found\" when going to a definition"
date: 2025-08-14
categories:
- Programming
tags:
- helix
- java
- lsp
- golang
---

If you follow this blog regularly, you know that I'm so happy with the combo [WezTerm](https://wezterm.org/index.html) (terminal emulator) and [Helix](https://helix-editor.com/) (editor).
Since I recently switched to a project that uses Java, I'm trying to configure Helix as my Java editor.

The default language server for Java is [jdtls](https://github.com/eclipse-jdtls/eclipse.jdt.ls) which can be installed easily on macOS by running:

```sh
brew install jdtls
```

```sh
hx --health java
Configured language servers:
  ✓ jdtls: /opt/homebrew/bin/jdtls
Configured debug adapter: None
Configured formatter: None
Tree-sitter parser: ✓
Highlight queries: ✓
Textobject queries: ✓
Indent queries: ✓
Tags queries: ✓
Rainbow queries: ✓
```

However, when I press `gd` (Go to definition) on a standard library or third-party class, Helix just returns "No definition found".
Even when starting Helix in verbose mode (`-vv`), the logs don't help much:

```log
2025-08-08T14:33:15.162 helix_lsp::transport [INFO] jdtls -> {"jsonrpc":"2.0","method":"textDocument/definition","params":{"position":{"character":17,"line":0},"textDocument":{"uri":"file:///Users/quantong/Code/personal/Exercism/java/diamond/src/main/java/DiamondPrinter.java"}},"id":2}
2025-08-08T14:33:15.259 helix_lsp::transport [INFO] jdtls <- {"jsonrpc":"2.0","id":2,"result":[]}
2025-08-08T14:33:15.259 helix_view::editor [DEBUG] editor error: No definition found.
```

It took me a few hours to realize that the client must enable `classFileContentsSupport` during initialization:

```toml
[language-server.jdtls.config]
extendedClientCapabilities.classFileContentsSupport = true
```

After enabling it, Helix finally revealed the real error:

```log
2025-08-08T16:36:55.027 helix_term::commands::lsp [WARN] discarding invalid or unsupported URI: unsupported scheme 'jdt' in URL jdt://contents/java.base/java.util/ArrayList.class?=diamond/%5C/opt%5C/homebrew%5C/Cellar%5C/openjdk%5C@21%5C/21.0.7%5C/libexec%5C/openjdk.jdk%5C/Contents%5C/Home%5C/lib%5C/jrt-fs.jar%60java.base=/javadoc_location=/https:%5C/%5C/docs.oracle.com%5C/en%5C/java%5C/javase%5C/21%5C/docs%5C/api%5C/=/%3Cjava.util(ArrayList.class
2025-08-08T16:36:55.027 helix_view::editor [DEBUG] editor error: No definition found.
```

So the problem is the response uses `jdt://` URL scheme which currently [Helix does not support](https://github.com/helix-editor/helix/blob/f4a433f855880667eb7888557c2e8a4b6d2c6dd3/helix-core/src/uri.rs?plain=1#L83):

```rust
fn convert_url_to_uri(url: &url::Url) -> Result<Uri, UrlConversionErrorKind> {
    if url.scheme() == "file" {
        url.to_file_path()
            .map(|path| Uri::File(helix_stdx::path::normalize(path).into()))
            .map_err(|_| UrlConversionErrorKind::UnableToConvert)
    } else {
        Err(UrlConversionErrorKind::UnsupportedScheme)
    }
}

impl fmt::Display for UrlConversionError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self.kind {
            UrlConversionErrorKind::UnsupportedScheme => {
                write!(
                    f,
                    "unsupported scheme '{}' in URL {}",
                    self.source.scheme(),
                    self.source
                )
            }
            UrlConversionErrorKind::UnableToConvert => {
                write!(f, "unable to convert URL to file path: {}", self.source)
            }
        }
    }
}
```

Fortunately, we can write a custom LSP server that sits between Helix and jdtls, translates `jdt://` to `file://` and displays the decompiled class file.

```go
cmd := exec.Command("jdtls", os.Args[1:]...)

serverStdin, err := cmd.StdinPipe()
if err != nil {
	return fmt.Errorf("connect to the stdin: %w", err)
}

serverStdout, err := cmd.StdoutPipe()
if err != nil {
	return fmt.Errorf("connect to the stdout: %w", err)
}

serverStderr, err := cmd.StderrPipe()
if err != nil {
	return fmt.Errorf("connect to the stderr: %w", err)
}

if err := cmd.Start(); err != nil {
	return fmt.Errorf("start: %w", err)
}
```

If the response scheme is `jdt://`, translate it to `file://`, write to a temp file, and return it back:

```go
if uri.Scheme == "jdt" {
	newID := id + 1
	classReq := map[string]any{
		"id":      newID,
		"jsonrpc": "2.0",
		"method":  "java/classFileContents",
		"params": map[string]any{
			"uri": definitionResult[0].URI,
		},
	}
	data, err := json.Marshal(classReq)
	if err != nil {
		fmt.Fprintln(os.Stderr, "[jdtls-wrapper]", err.Error())
	}

	pending[newID] = func(resp *jdtResponse) {
		fmt.Fprintln(os.Stderr, "[jdtls-wrapper]", string(resp.Result))
		result, err := strconv.Unquote(string(resp.Result))
		if err != nil {
			fmt.Fprintln(os.Stderr, "[jdtls-wrapper]", err.Error())
		}

		result = strings.ReplaceAll(result, `\n`, "\n")
		result = strings.ReplaceAll(result, `\t`, "\t")
		tmpFileName := "/tmp" + strings.TrimSuffix(uri.Path, ".class") + ".java"
		targetURI := "file://" + tmpFileName
		m[targetURI] = uri.String()
		if err := os.MkdirAll(filepath.Dir(tmpFileName), 0755); err != nil {
			fmt.Fprintln(os.Stderr, "[jdtls-wrapper]", err.Error())
		}
		if err := os.WriteFile(tmpFileName, []byte(result), 0400); err != nil {
			fmt.Fprintln(os.Stderr, "[jdtls-wrapper]", err.Error())
		}
		targetRange := Range{
			Start: Position{
				Line:      definitionResult[0].Range.Start.Line,
				Character: definitionResult[0].Range.Start.Character,
			},
			End: Position{
				Line:      definitionResult[0].Range.End.Line,
				Character: definitionResult[0].Range.End.Character,
			},
		}
		stdResp := &stdResponse{
			Jsonrpc: "2.0",
			ID:      int64(id),
			Result: []stdResult{
				{
					TargetURI:            targetURI,
					TargetRange:          targetRange,
					TargetSelectionRange: targetRange,
				},
			},
		}

		data, err := json.Marshal(stdResp)
		if err == nil {
			fmt.Fprintf(os.Stdout, "Content-Length: %d\r\n\r\n%s", len(data), data)
		}
	}

	fmt.Fprintf(serverStdin, "Content-Length: %d\r\n\r\n%s", len(data), data)
}
```

More details can be found [here](https://github.com/quantonganh/jdtls-wrapper).
