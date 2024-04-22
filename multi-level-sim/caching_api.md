# Caching API

## Sketch

Let's do a ultra simple pipeline that just consists of one python function without any file IO.
We just want to cache this key based on its args to avoid executing the rule multiple times if unnecessary.
We also want a simple CLI to fetch this key

```python
# A build rule with no dependencies
class String(Key)[str]:
  arg1: int
  arg2: str
  dependencies: None
  def rule(self, dependencies) -> str:
    return f"arg1: {self.arg1}, arg2: {self.arg2} the string"

# A build rule that just produces a path to a file on disk
class File(Key)[List[pathlib.Path]]:
  root_dir: Path
  def rule(self, dependencies) -> List[Path]:  # returning a List[Path] is implicitly saying that some files were "produced" by this build rule, so the runtime needs to hash those files + paths
    return [self.root_dir / "file1.txt", self.root_dir / "file2.txt"]

# A build rule that depends on files via a key

String(1, "string1").run(Path.cwd() / "cache")
String(1, "string1").run(Path.cwd() / "cache")  # shouldn't rerun the rule
String(2, "string2").run(Path.cwd() / "cache")  # should rerun the rule
```

Will we have support for programmatically generating keys?
What should happen if a file is randomly referenced in the rule()? That should cause a dependency track actually? But we can't identify such things happening in general in Python.
