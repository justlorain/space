---
title: "Add License Headers to Your Code Files"
description: "As developers, we often need to ensure that our code complies with the appropriate licenses to protect our intellectual property."
publishDate: "28 Aug 2023"
tags: ["tutorial", "opensource", "programming", "go"]
---

## Introduction

As developers, we often need to ensure that our code complies with the appropriate licenses to protect our intellectual property.

This article introduces a more powerful license statement management tool called [**NWA**](https://github.com/B1NARY-GR0UP/nwa). It helps you effortlessly **add** license headers to your code files and also **check**, **update**, and **remove** existing license statements, ensuring the legality and compliance of your code.

## What Is a License Statement?

A license statement (license header) is a comment section at the top of a source code file that declares the licensing information for the code. It typically includes copyright information, license type, and other relevant legal statements. License statements help protect the rights of the code's author and also help other developers understand how they can use the code.

If you've ever looked at code files from open-source projects, you've probably noticed that they clearly mark their license statements within the code files. For example:

- **kubernetes**


![k8s-example](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/01i5o5pftic2ltzjn6jk.png)

- **spring-boot**


![spring-boot-example](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9fp8eqqq28ueyu71gas4.png)

Additionally, some open-source projects use tools like GitHub Actions to check if your code has the correct license header when you submit a pull request:

- **hertz**


![hertz-example](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/eam49w0uotul2mw0gbow.png)

## License Header Adding Tools

Usually, we need to display license statements in the form of comments at the beginning of each code file, similar to what 'kubernetes' and 'spring-boot' did in the previous section. However, when you are working on a large project, the number of code files can become extensive, making manually adding license headers a **tedious and painful** task.

This is where license header adding tools come in handy. One commonly used tool is the [addlicense](https://github.com/google/addlicense) project under the 'google' organization. 'addlicense' allows you to add license statements to specified files via command-line interaction. However, 'addlicense' has limited functionality; it only adds license headers to files. Some issues raised for 'addlicense' include:

- Request for support for line comments (`//`) instead of only block comments (`/**/`) [#130](https://github.com/google/addlicense/issues/130)
- Request to add a feature for updating license statements [#58](https://github.com/google/addlicense/issues/58)
- Request for customizable license statement checking logic [#56](https://github.com/google/addlicense/issues/56)

Unfortunately, issue #58 is not supported due to concerns about adding complexity to 'addlicense', issue #130 is marked as unsupported, and issue #56 is currently awaiting review.

NWA's development took these issues into consideration and provided solutions. It also supports more operations related to license statements, such as removal. You can see how NWA addresses these issues and get started quickly in the [Usage](#usage) section.

## Installation

### Installation via Go (if you have Go installed)

```shell
go install github.com/B1NARY-GR0UP/nwa@latest
```

Execute the following command to verify a successful installation:

```shell
nwa --version
```

### Installation via Docker (if you don't have Go installed)

1. **Installation**

Get NWA's Docker image directly:

```shell
docker pull ghcr.io/b1nary-gr0up/nwa:main
```

Alternatively, you can build the image from source, for example:

```shell
docker build -t ghcr.io/b1nary-gr0up/nwa:main .
```

2. **Verify Installation**

```shell
docker run -it ghcr.io/b1nary-gr0up/nwa:main --version
```

3. **Mount the directory you want NWA to operate on to `/src`. Here's an example, but you can learn more about NWA's usage in the [Usage](#usage) section:**

```shell
docker run -it -v ${PWD}:/src ghcr.io/b1nary-gr0up/nwa:main add -c "RHINE LAB.LLC." -y 2077 .
```

## Usage

To help you get started with NWA's functionality, developers have provided [**nwa-examples**](https://github.com/rainiring/nwa-examples) to help you learn how to use NWA in a practical way. You can clone this project and follow the tutorial or simply refer to this guide and use nwa-examples for more usage examples.

NWA is a command-line tool built on [cobra](https://github.com/spf13/cobra). Here's an overview of NWA's commands:

```shell
Usage:         
  nwa [command]

Common Mode Commands:
  add         add license headers to files
  check       check license headers of files
  remove      remove license headers of files
  update      update license headers of files

Config Mode Commands:
  config      edit the files according to the configuration file

Additional Commands:
  help        Help about any command

Flags:
  -h, --help      help for nwa
  -v, --version   version for nwa

Use "nwa [command] --help" for more information about a command.
```

**Command List:**

- **Add**: Add license headers to files.
- **Remove**: Remove license headers from files.
- **Update**: Update license headers in files.
- **Check**: Check license headers in files.
- **Config**: Edit license headers in files using a configuration file.

### Add - Add License Headers to Files

#### Usage

```shell
nwa add [flags] path...
```

Use the `add` command after `nwa` to add license headers. `[flags]` allows you to specify optional flags, and `path...` lets you specify one or more file paths to add license headers to.

#### Example

```shell
nwa add -l apache -c "RHINE LAB.LLC." -y 2077 ./server ./utils/bufferpool
```

In the above example, the command adds the following to all files in the relative paths `./server` and `./utils/bufferpool`:

- License type: `Apache 2.0`
- Copyright holder: `RHINE LAB.LLC.`
- Copyright year: `2077`

NWA generates the corresponding license statement for each file based on its type, e.g., `.py` files use `#` comments, and `.go` files use `//` comments. If your file type is not supported by NWA, you can:

- Specify a template file using the `-t` (`--tmpl`) flag.
- Contribute to NWA by creating an issue or pull request.

NWA will also provide log

outputs indicating files that already have license headers or are not allowed for editing (such as some generated code files).

#### Flags

The `add` command supports the following flags:

| Short Flag | Long Flag   | Default                            | Description        |
| ---------- | ----------- | ---------------------------------- | ------------------ |
| -c         | --copyright | `<COPYRIGHT HOLDER>`               | Copyright holder   |
| -l         | --license   | `apache`                           | License type       |
| -m         | --mute      | `false` (unspecified)              | Mute mode          |
| -s         | --skip      | `[]`                               | Skipped file paths |
| -t         | --tmpl      | `""`                               | Template file path |
| -y         | --year      | `time.Now().Year()` (current year) | Copyright year     |
| -h         | --help      | None                               | Help               |

Most flags are self-explanatory. Notable flags are `-s` (`-skip`) and `-t` (`-tmpl`):

- `-s`'s default value is an empty array, so you can use zero or more `-s` flags to specify file paths to skip. This path supports [doublestar](https://github.com/bmatcuk/doublestar) syntax, making it very flexible. For example:

```shell
nwa add -s **.py -s /example/**/*.txt -c Lorain .
```

- `-t` allows you to specify a template file path. NWA reads the content of this template file and adds its content to the files that require license headers.

**Note: If you specify a template, NWA won't generate license headers based on file types, and it will ignore `-c`, `-l`, and `-y` configurations.**

Here's an example of a template file (`example-tmpl.txt`):

```text
// Copyright 2077 RHINE LAB.LLC.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.
//
```

For more usage examples, refer to [nwa-examples](https://github.com/rainiring/nwa-examples).

### Remove - Remove License Headers from Files

#### Usage

```shell
nwa remove [flags] path...
```

Use the `remove` command after `nwa` to remove license headers. `[flags]` allows you to specify optional flags, and `path...` lets you specify one or more file paths to remove license headers from.

#### Example

```shell
nwa remove -l mit -c "RHINE LAB.LLC." -s **.py pkg
```

In the above example, the command removes the following from all **non-Python** files in the relative path `pkg`:

- License type: `MIT`
- Copyright holder: `RHINE LAB.LLC.`
- Copyright year: `2023`

NWA provides log outputs indicating files without license headers or files that are not editable.

#### Flags

The `remove` command supports the following flags:

| Short Flag | Long Flag   | Default                            | Description        |
| ---------- | ----------- | ---------------------------------- | ------------------ |
| -c         | --copyright | `<COPYRIGHT HOLDER>`               | Copyright holder   |
| -l         | --license   | `apache`                           | License type       |
| -m         | --mute      | `false` (unspecified)              | Mute mode          |
| -s         | --skip      | `[]`                               | Skipped file paths |
| -t         | --tmpl      | `""`                               | Template file path |
| -y         | --year      | `time.Now().Year()` (current year) | Copyright year     |
| -h         | --help      | None                               | Help               |

**Note: If you specify a template, NWA won't generate license headers based on file types, and it will ignore `-c`, `-l`, and `-y` configurations.**

For more usage examples, refer to [nwa-examples](https://github.com/rainiring/nwa-examples).

### Update - Update License Headers in Files

#### Usage

```shell
nwa update [flags] path...
```

Use the `update` command after `nwa` to update license headers. `[flags]` allows you to specify optional flags, and `path...` lets you specify one or more file paths to update license headers in.

#### Example

```shell
nwa update -l apache -c "BINARY Members" .
```

In the above example, the command updates the license headers in all files in the current path with the following:

- License type: `Apache 2.0`
- Copyright holder: `BINARY Members`
- Copyright year: `2023`

The previous copyright statement is replaced, regardless of its content.

**Note: Update is essentially a combination of Remove and Add operations.**

The `update` command recognizes the content before the first empty line in a file as the license header (with the exception of shebang lines). NWA generates a new license statement based on the specified flags and replaces the existing one.

If the content before the first empty line in your file includes something other than the license statement, it's recommended to use the `remove` and `add` operations separately to achieve the same effect. This approach, along with the template file flag, can lead to better removal results in the `remove` operation.

#### Flags

The `update` command supports the following flags:

| Short Flag | Long Flag   | Default                            | Description        |
| ---------- | ----------- | ---------------------------------- | ------------------ |
| -c         | --copyright | `<COPYRIGHT HOLDER>`               | Copyright holder   |
| -l         | --license   | `apache`                           | License type       |
| -m         | --mute      | `false` (unspecified)              | Mute mode          |
| -s         | --skip      | `[]`                               | Skipped file paths |
| -t         | --tmpl      | `""`                               | Template file path |
| -y         | --year      | `time.Now().Year()` (current year) | Copyright year     |
| -h         | --help      | None                               | Help               |

**Note: If you specify a template, NWA won't generate license headers based on file types, and it will ignore `-c`, `-l`, and `-y` configurations.**

For more usage examples, refer to [nwa-examples](https://github.com/rainiring/nwa-examples).

### Check - Check License Headers in Files

#### Usage

```shell
nwa check [flags] path...
```

Use the `check` command after `nwa` to check license headers. `[flags]` allows you to specify optional flags, and `path...` lets you specify one or more file paths to check license headers in.

#### Example

```shell
nwa check --tmpl tmpl.txt ./client
```

In the above example, the command checks if license headers in all files within the `./client` folder match the content specified in the `tmpl.txt` template file.

After checking, NWA provides log outputs indicating the result of the check. Here's a sample output:

```text
time="2023-08-25T23:01:49+08:00" level=info msg="skip file" path=README.md
time="2023-08-25T23:01:49+08:00" level=info msg="file has a matched header" path="dirA\\fileA.go"                
time="2023-08-25T23:01:49+08:00" level=info msg="file does not have a matched header" path="dirB\\dirC\\fileC.go"
time="2023-08-25T23:01:49+08:00" level=info msg="file has a matched header" path="dirB\\fileB.go"                
time="2023-08-25T23:01:49+08:00" level=info msg="file does not have a matched header" path=main.go 
```

#### Flags

The `check` command supports the following flags:

| Short Flag | Long Flag   | Default                            | Description        |
| ---------- | ----------- | ---------------------------------- | ------------------ |
| -c         | --copyright | `<COPYRIGHT HOLDER>`               | Copyright holder   |
| -l         | --license   | `apache`                           | License type       |
| -m         | --mute      | `false` (unspecified)              | Mute mode          |
| -s         | --skip      | `[]`                               | Skipped file paths |
| -t         | --tmpl      | `""`                               | Template file path |
| -y         | --year      | `time.Now().Year()` (current year) | Copyright year     |
| -h         | --help      | None                               | Help               |

**Note: Avoid using the `-m` flag with the `check` command, as it will prevent NWA from outputting the check results.**

For more usage examples, refer to [nwa-examples](https://github.com/rainiring/nwa-examples).

### Config - Edit License Headers via Configuration File

#### Usage

```shell
nwa config [flags] path
```

Use the `config` command after `nwa` to edit license headers based on a configuration file. `[flags]` allows you to specify optional flags, and `path` is **required** to specify the path to the configuration file.

#### Example

```shell
nwa config config.yaml
```

In the above example, the command reads the `config.yaml` configuration file and edits license headers in specified files according to its content. The structure of the configuration file is provided below.

#### Flags

Since all configuration is in the configuration file, there's only one help flag.

| Short Flag | Long Flag | Default | Description |
| ---------- | --------- | ------- | ----------- |
| -h         | --help    | None    | Help        |

#### Configuration File

Here's an example of a complete configuration file (`config.yaml`):

```yaml
nwa:
  cmd: "add"                        # Default: "add". Options: "add", "check", "remove", "update"
  holder: "RHINE LAB.LLC."          # Default: "<COPYRIGHT HOLDER>"
  year: "2077"                      # Default: Current Year
  license: "apache"                 # Default: "apache"
  mute: false                       # Default: false (unspecified)
  path: ["server", "client", "pkg"] # Default: []
  skip: ["**.py"]                   # Default: []
  tmpl: "nwa.txt"                   # Default: ""
```

If you don't specify certain fields in the configuration file, NWA will use default values. For example, if you don't specify the license type, Apache 2.0 will be used by default.

**Note: If you set the `tmpl` field, NWA will ignore the `holder`, `year`, and `license` fields.**

For more usage examples, refer to [nwa-examples](https://github.com/rainiring/nwa-examples).

## Conclusion

The above content introduces **NWA — A Powerful License Header Management Tool**.

If you find this tool helpful, please consider giving our project a star!

If you notice any errors or have questions, feel free to leave a comment or send a message.

## References

- https://github.com/B1NARY-GR0UP/nwa
- https://github.com/rainiring/nwa-examples
- https://github.com/google/addlicense
- https://github.com/bmatcuk/doublestar
- https://github.com/spf13/cobra