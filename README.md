

Quick example
Python
import json_repair

bad_json = '{"users":[{"name":"Ada","role":"admin",}],"ok":true'
decoded_object = json_repair.loads(bad_json)

# {'users': [{'name': 'Ada', 'role': 'admin'}], 'ok': True}
If json_repair saves you time, [star the repository](https://github.com/[Your Username]/json_repair) so more people can find it.

Motivation
Some LLMs are a bit iffy when it comes to returning well-formed JSON data. Sometimes they skip a parenthesis, and sometimes they add conversational filler, because that is simply how LLMs operate. Luckily, the mistakes LLMs make are simple enough to be fixed without destroying the underlying content.

I needed a reliable, lightweight Python package capable of fixing this problem consistently in my own automated workflows. I adopted and enhanced this codebase to ensure I had a robust tool that does exactly what I need: salvage broken JSON without bloated dependencies.

Supported use cases
Fixing Syntax Errors in JSON
Missing quotes, misplaced commas, unescaped characters, and incomplete key-value pairs.

Missing quotation marks, improperly formatted values (true, false, null), and corrupted key-value structures.

Repairing Malformed JSON Arrays and Objects
Incomplete or broken arrays/objects are fixed by adding necessary elements (e.g., commas, brackets) or default values (null, "").

The library processes JSON that includes extra non-JSON characters like comments or improperly placed text, cleaning them up while maintaining valid structure.

Auto-Completion for Missing JSON Values
Automatically completes missing values in JSON fields with reasonable defaults (like empty strings or null), ensuring validity.

How to use
Install the library with pip:

Bash
pip install json-repair
Then you can use it in your code like this:

Python
from json_repair import repair_json

good_json_string = repair_json(bad_json_string)
# If the string was super broken this will return an empty string
You can use this library to completely replace json.loads():

Python
import json_repair

decoded_object = json_repair.loads(json_string)
Or just return the parsed objects directly:

Python
import json_repair

decoded_object = json_repair.repair_json(json_string, return_objects=True)
Avoid this antipattern
Some developers adopt the following pattern:

Python
obj = {}
try:
    obj = json.loads(string)
except json.JSONDecodeError as e:
    obj = json_repair.loads(string)
    # ...
This is wasteful because json_repair already does that strict json.loads() check for you by default. The normal flow is:

Try the built-in json.loads() / json.load() first.

If that succeeds, return the decoded object.

If that fails, run the repair parser.

Use the default call unless you explicitly want to skip that initial validation step:

Python
import json_repair

decoded_object = json_repair.loads(json_string)
Read JSON from a file or file descriptor
JSON repair also provides a drop-in replacement for json.load():

Python
import json_repair

try:
    file_descriptor = open(fname, 'rb')
except OSError:
    pass

with file_descriptor:
    decoded_object = json_repair.load(file_descriptor)
And another method to read directly from a file path:

Python
import json_repair

try:
    decoded_object = json_repair.from_file(json_file)
except OSError:
    pass
except IOError:
    pass
Keep in mind that the library will not catch any IO-related exceptions; those will need to be managed by your code.

Non-Latin characters
When working with non-Latin characters (such as Chinese, Japanese, or Korean), you need to pass ensure_ascii=False to repair_json() in order to preserve the original characters in the output.

Here is an example:

Python
repair_json("{'test_chinese_ascii':'统一码'}")
# Returns: {"test_chinese_ascii": "\u7edf\u4e00\u7801"}

repair_json("{'test_chinese_ascii':'统一码'}", ensure_ascii=False)
# Returns: {"test_chinese_ascii": "统一码"}
JSON dumps parameters
In general, repair_json accepts all parameters that json.dumps accepts and passes them through (for example, indent).

Performance considerations
By default, json_repair first tries the standard-library JSON loader and only falls back to the repair parser when strict JSON parsing fails.

If you already know the input is invalid JSON and want to skip that initial validation step, pass skip_json_loads=True:

Python
from json_repair import repair_json

good_json_string = repair_json(bad_json_string, skip_json_loads=True)
This is an explicit tradeoff:

Default behavior: Validate with stdlib JSON first, then repair only if needed.

skip_json_loads=True: Skip the validation fast path and go straight to the repair parser.

Important: skip_json_loads=True is only for inputs you already know are invalid. If you force already-valid JSON through the repair parser, json_repair may still "repair" it and can change the resulting structure or values. If you need valid JSON to be preserved as-is, keep skip_json_loads=False.

Rules of thumb for optimization:

Setting return_objects=True will always be faster because the parser returns an object already and does not have to serialize that object back to JSON.

skip_json_loads=True is faster only if you 100% know that the string is not a valid JSON.

If you are having issues with escaping, pass the string as a raw string, like this: r"string with escaping\"".

When to use your own JSON library
If you want non-stdlib JSON semantics or a different performance profile, use your preferred JSON library yourself instead of expecting json_repair to switch parsers automatically. orjson is a common example, and the same pattern applies to any other JSON library.

orjson first, json_repair only as a fallback:

Python
import json_repair
import orjson

try:
    decoded_object = orjson.loads(json_string)
except orjson.JSONDecodeError:
    decoded_object = json_repair.loads(json_string, skip_json_loads=True)
Strict mode
By default, json_repair does its best to “fix” input, even when the JSON is far from valid. In some scenarios, you want the opposite behavior and need the parser to error out instead of repairing. Pass strict=True to repair_json, loads, load, or from_file to enable that mode:

Python
from json_repair import repair_json

repair_json(bad_json_string, strict=True)
The CLI exposes the same behavior with json_repair --strict input.json (or piping data via stdin).

In strict mode, the parser raises ValueError as soon as it encounters structural issues such as duplicate keys, missing : separators, or empty keys/values introduced by stray commas. Strict mode still honors skip_json_loads=True.

Schema-guided repairs
Schema-guided repairs are currently considered in beta. Bugs are to be expected.

You can guide repairs with a JSON Schema (or a Pydantic v2 model). When enabled, the parser will:

Fill missing values (defaults, required fields).

Coerce scalars where safe (e.g., "1" → 1 for integer fields, and "yes"/"no"/1/0 for booleans).

Drop properties/items that the schema disallows.

Schema mode can be selected with schema_repair_mode:

standard (default): Applies standard existing schema-guided behavior.

salvage (enhanced): Includes standard behavior but also applies more aggressive fixes.

salvage feature: Drops invalid array items when individual items cannot be repaired.

salvage feature: Maps arrays to objects by property order when schema/object shape is unambiguous.

salvage feature: Unwraps a root single-item array to an object when the root schema expects an object ([{...}] -> {...}).

salvage feature: Fills missing required fields only when a safe value can be inferred (default, const, first enum, or empty array/object).

Install the optional dependencies:

Bash
pip install 'json-repair[schema]'
Example usage:

Python
from json_repair import repair_json

schema = {
    "type": "object",
    "properties": {"value": {"type": "integer"}},
    "required": ["value"],
}

repair_json('{"value": "1"}', schema=schema, return_objects=True)
Pydantic v2 model example:

Python
from pydantic import BaseModel, Field
from json_repair import repair_json

class Payload(BaseModel):
    value: int
    tags: list[str] = Field(default_factory=list)

repair_json(
    '{"value": "1", "tags": }',
    schema=Payload,
    skip_json_loads=True,
    return_objects=True,
)
Use json_repair with streaming
Sometimes you are streaming data and want to repair the partial JSON coming from it. Normally this won't work, but you can pass stream_stable to repair_json() or loads() to make it work:

Python
stream_output = repair_json(stream_input, stream_stable=True)
More integration examples
If you want copy-paste examples for real applications, see the examples/ directory in this repository:

repair_llm_output.py: Repairs markdown-wrapped or prose-wrapped model output.

pydantic_schema.py: Uses a Pydantic v2 model as schema guidance.

stream_stable.py: Keeps partial JSON stable during streaming.

fastapi_app.py: Drops the repair step into a FastAPI endpoint.

Use json_repair from CLI
Install the library for command-line use with:

Bash
pipx install json-repair
To see all available options:

Bash
json_repair -h
Adding to requirements
Please pin this library only on the major version!

I follow strict semantic versioning. There will be frequent updates and no breaking changes in minor and patch versions. Specify the package name followed by the major version and a wildcard for minor and patch versions:

Plaintext
json_repair==0.*
How it works
This module parses the JSON file following the standard BNF definition. If something is wrong (a missing parenthesis or quotes, for example), it uses simple heuristics to fix the JSON string:

Adds missing parentheses if the parser believes that the array or object should be closed.

Quotes strings or adds missing single quotes.

Adjusts whitespaces and removes line breaks.

If you encounter a missing corner case, please open an issue or push a PR to my repository.

Contributing
If you want to contribute, start with CONTRIBUTING.md.

How to develop
Use uv to set up the dev environment and run tooling:

Bash
uv sync --group dev
uv run pre-commit run --all-files
uv run pytest
Make sure that the Github Actions running after pushing a new commit do not fail.

How to release
You will need owner access to this repository.

Edit pyproject.toml and update the version number appropriately using semver notation.

Commit and push all changes to the repository before continuing, or the next steps will fail.

Run python -m build.

Create a new release in Github. Create the new tag, same as the one in the build configuration.

Once the release is created, a new Github Actions workflow will start to publish on Pypi.

Docs demo API deployment
The docs site is deployed by GitHub Pages.

After a successful Pages deployment on main, the workflow uploads docs/app.py to your server environment.

Requires repository Actions secret: PYTHONANYWHERE_API_TOKEN.
