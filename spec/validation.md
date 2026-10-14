# Rheon v1 — Reference Validation

Both implementations do the same thing in the same order:

1. parse JSON
2. require top-level `v`, `source`, `type`, `event`
3. require `v == 1`
4. check `type` / `event` format
5. look up `type:event` in the registry
6. require the schema's `required` fields
7. otherwise warn / ignore (unknown signal → continue; unknown fields → allowed)

⚠️ **Step 0 is decoding, and it has the same answer as step 7.** A line that is
not valid UTF-8 is not valid NDJSON (§2.1). Report it and **read the next line**
— do not end the stream, which is what a `read_line`-into-a-string loop does by
default in several languages, and which makes a truncated stream look complete.

Both sketches below take an already-decoded `line`. A reader that owns the bytes
should decode per line (`Vec<u8>` → lossy `String`, or Python's
`errors="replace"`), not per stream.

## Python

```python
import json

def is_valid_ident(s: str) -> bool:
    return all(c.islower() or c.isdigit() or c == "-" for c in s)

def validate_line(line: str, registry: dict) -> None:
    v = json.loads(line)                                   # 1
    for field in ("v", "source", "type", "event"):        # 2
        if field not in v:
            raise ValueError(f"missing field: {field}")
    if v["v"] != 1:                                        # 3
        raise ValueError(f"unsupported version: {v['v']}")
    t, e = v["type"], v["event"]                           # 4
    if not (isinstance(t, str) and isinstance(e, str)):
        raise ValueError("type/event must be strings")
    if not (is_valid_ident(t) and is_valid_ident(e)):
        raise ValueError(f"invalid type/event format: {t}:{e}")
    sig = registry.get("signals", {}).get(f"{t}:{e}")      # 5
    if sig is None:
        print(f"warning: unknown signal: {t}:{e}")         # 7 (continue)
        return
    for req in sig.get("schema", {}).get("required", []):  # 6
        if req not in v:
            raise ValueError(f"missing required field: {req}")
```

## Rust (sketch)

```rust
fn validate_line(line: &str, reg: &Registry) -> Result<(), String> {
    let v: serde_json::Value = serde_json::from_str(line).map_err(|e| e.to_string())?;
    for f in ["v", "source", "type", "event"] {
        if v.get(f).is_none() { return Err(format!("missing field: {f}")); }
    }
    if v["v"] != 1 { return Err("unsupported version".into()); }
    let (t, e) = (v["type"].as_str().ok_or("type")?, v["event"].as_str().ok_or("event")?);
    if !is_valid_ident(t) || !is_valid_ident(e) { return Err(format!("invalid format: {t}:{e}")); }
    match reg.signals.get(&format!("{t}:{e}")) {
        None => { eprintln!("warning: unknown signal: {t}:{e}"); Ok(()) }   // continue
        Some(s) => {
            for req in &s.required {
                if v.get(req).is_none() { return Err(format!("missing required: {req}")); }
            }
            Ok(())
        }
    }
}

fn is_valid_ident(s: &str) -> bool {
    s.chars().all(|c| c.is_ascii_lowercase() || c.is_ascii_digit() || c == '-')
}
```
