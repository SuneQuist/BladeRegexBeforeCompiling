# BladeRegexBeforeCompiling

An experimental **Laravel Blade precompiler and lightweight parsing utility** for inspecting Blade templates before they are fully compiled.

The project was created while working with **Laravel / Roots Acorn / Sage** and exploring how Blade templates could be intercepted and analyzed earlier in the compilation process.

Rather than operating on the final compiled PHP, the package registers a Blade precompiler and receives the original template source before Blade performs its normal compilation.

## Why I Built It

I wanted access to the structure of Blade templates before compilation so I could inspect constructs such as:

```blade
{{ $variable }}

{!! $rawHtml !!}

<div class="{{ $class }}">
    ...
</div>
```

Working on the original template makes it possible to reason about Blade-specific syntax that may be transformed or lost later in the compilation process.

This project therefore became an experiment in two areas:

- hooking into the Blade compilation pipeline
- parsing nested or paired syntax using regular expressions together with stack-based matching

## How It Works

The package is loaded through a Roots Acorn service provider.

During application startup, it obtains the Blade compiler and registers a custom precompiler callback.

```text
Blade template
      │
      ▼
Laravel / Acorn
      │
      ▼
Blade compiler
      │
      ▼
Custom precompiler
      │
      ▼
IffyCompiler
      │
      ▼
Pattern / structure analysis
      │
      ▼
Original Blade source returned
      │
      ▼
Normal Blade compilation continues
```

The important part is that the custom logic runs **before normal Blade compilation**.

The precompiler currently leaves the source unchanged after analyzing it, making the project suitable for experimentation without replacing Blade's normal compiler behavior.

## Blade Precompiler

The integration is handled by `IffyBladeCompiler`.

Conceptually, it registers logic equivalent to:

```php
$bladeCompiler->precompiler(function ($html) {
    new IffyCompiler($html);

    return $html;
});
```

The raw Blade template can therefore be inspected before Laravel compiles directives, expressions, and Blade echo syntax into PHP.

## Pattern Matching

The current prototype looks for several types of Blade/template syntax, including:

```text
HTML attributes

Raw variables

Blade raw echo:
{!! ... !!}

Blade echo:
{{ ... }}

Triple-brace echo:
{{{ ... }}}
```

These patterns are passed into the custom `IffyUtils::inbetween()` function.

## Stack-Based Delimiter Matching

The most interesting part of the project is the `inbetween()` utility.

A simple regular expression can often find an opening and closing sequence, but matching nested structures correctly becomes more difficult.

Instead, this implementation:

1. finds all opening matches
2. finds all closing matches
3. records their positions in the source text
4. combines and sorts the matches by position
5. uses a stack-like structure to associate closing delimiters with the appropriate opening delimiter
6. extracts the matched content and its offsets

Conceptually:

```text
Input:

    {{ outer {{ inner }} value }}


Matches:

    OPEN
        OPEN
        CLOSE
    CLOSE


Stack processing:

    OPEN outer
    OPEN inner
    CLOSE -> matches inner
    CLOSE -> matches outer
```

The resulting objects contain information such as:

```text
opening delimiter
closing delimiter

full content
inner content

content length
inner content length

opening position
closing position

inner-content start
inner-content end
```

This makes the utility more useful than a single flat regular-expression match when dealing with nested or repeated delimiters.

## Example

Given an opening and closing expression:

```php
$result = IffyUtils::inbetween(
    $template,
    ['/{!!/', '/!!}/']
);
```

the utility can identify sections such as:

```blade
{!! $content !!}
```

and return metadata describing where the match occurred and what was contained between the delimiters.

The same mechanism can be reused with other opening/closing expressions.

## Configuration

Pattern pairs can also be stored in the package configuration.

The repository currently includes examples for:

```php
'closures' => [
    ['/((\w|\_|\-)+[^\s])=\"/', '/(.+?)\"/'],
    ['/\$/', '/\s/'],
    ['/{!!/', '/!!}/'],
    ['/{{/', '/}}/'],
    ['/{{{/', '/}}}/'],
],
```

This makes it possible to experiment with different structures without hard-coding every pattern directly into the parsing logic.

## Project Structure

```text
BladeRegexBeforeCompiling/
│
├── config/
│   └── config.php
│
├── src/
│   ├── Bin/
│   │   └── IffyUtils.php
│   │
│   ├── Blade/
│   │   └── Compiler/
│   │       ├── IffyBladeCompiler.php
│   │       └── IffyCompiler.php
│   │
│   ├── Providers/
│   │   └── IffyServiceProvider.php
│   │
│   └── Iffy.php
│
├── composer.json
└── README.md
```

## Installation for Local Development

This project was designed as a local development package rather than a published Composer package.

Add the repository as a Composer path repository:

```json
{
    "repositories": [
        {
            "type": "path",
            "url": "/path/to/BladeRegexBeforeCompiling",
            "options": {
                "symlink": true
            }
        }
    ]
}
```

Then require the package:

```json
{
    "require": {
        "sune/iffy": "@dev"
    }
}
```

Update Composer:

```bash
composer update
composer dump-autoload
```

When using Roots Acorn, package discovery may also be refreshed with:

```bash
wp acorn package:discover
```

The package service provider is registered through Composer/Acorn package metadata.

## Current State

This repository should be considered a **prototype / learning experiment**, not a production-ready Blade extension.

The current compiler analyzes matching structures and outputs the results for inspection rather than transforming the Blade source.

For example, the current implementation uses:

```php
var_dump($list);
```

to display matches generated by the parser.

That was sufficient for exploring Blade's precompiler behavior and validating the matching algorithm.

## Potential Uses

The same architecture could be extended into tooling such as:

```text
Blade template linting

Static analysis

Detection of raw Blade output

Template security checks

Custom Blade syntax

Template transformation

Developer diagnostics

Source-to-source preprocessing
```

A particularly interesting extension would be a security-oriented Blade analyzer that warns about potentially dangerous template constructs before the templates are compiled.

For example:

```text
resources/views/profile.blade.php

[WARNING] Raw Blade output detected

    {!! $userContent !!}

Review whether the value can contain
untrusted or user-controlled HTML.
```

## What I Learned

The project gave me practical experience with:

- Laravel Blade internals
- compiler/precompiler hooks
- Roots Acorn service providers
- Composer path repositories
- PSR-4 autoloading
- PHP regular expressions
- offset-based string processing
- stack-based matching
- nested delimiter parsing
- framework extension points

It also reinforced an important distinction between simply matching text with regular expressions and building enough surrounding logic to reason about nested structures.

## Scope

The parser is intentionally lightweight.

It is not intended to be a complete HTML, PHP, or Blade parser and does not implement a formal grammar or abstract syntax tree.

Instead, it was built as an experiment in intercepting Blade source and performing structured analysis on selected patterns before the normal Blade compiler takes over.
