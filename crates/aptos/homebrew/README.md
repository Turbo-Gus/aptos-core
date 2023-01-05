# Homebrew Aptos

This is an in-depth overview of homebrew and the aptos formula. In this guide we will go over each section of the homebrew formula as well as steps to implement changes in the future.

## Quick guide

- [Formula in Homebrew Github](https://github.com/Homebrew/homebrew-core/blob/master/Formula/aptos.rb)
- [Aptos 1.0.3 New Formula PR for Github](https://github.com/Homebrew/homebrew-core/pull/119832)

### Getting started
Copy the `aptos.rb` file to your homebrew folders directory. On Mac M1, this will most likely be:

```bash
/opt/homebrew/Library/Taps/homebrew/homebrew-core/Formula
```


### Development

After you've copied `aptos.rb` to your local homebrew formulas directory, you can modify it and use the commands below for testing.

```bash
# On Mac M1, homebrew formulas are located locally at
/opt/homebrew/Library/Taps/homebrew/homebrew-core/Formula

# Before submitting changes run
brew audit --new-formula aptos      # For new formula
brew audit aptos --strict --online
brew install aptos
brew test aptos

# For debugging issues during the installation process you can do
brew install aptos --interactive    # Interactive, gives you access to the shell
brew install aptos -d               # Debug mode

# Livecheck
brew livecheck --debug aptos
```

### Committing changes

Once you have audited and tested your brew formula using the commands above, make sure you:

1. Commit your changes to `aptos-core` in `crates/aptos/homebrew`
2. Fork the homebrew core repo, more details can be found [here](https://docs.brew.sh/How-To-Open-a-Homebrew-Pull-Request#formulae-related-pull-request)
3. Create a PR on the [Homebrew Core](https://github.com/Homebrew/homebrew-core/pulls) repo with your changes

## Aptos.rb structure overview

### Header

```ruby
class Aptos < Formula
  desc "Layer 1 blockchain built to support fair access to decentralized assets for all"
  homepage "https://aptoslabs.com/"
  url "https://github.com/aptos-labs/aptos-core/archive/refs/tags/aptos-cli-v1.0.3.tar.gz"
  sha256 "670bb6cb841cb8a65294878af9a4f03d4cba2a598ab4550061fed3a4b1fe4e98"
  license "Apache-2.0"
  ...
```

### Bottles

[Bottles](https://docs.brew.sh/Bottles#pour-bottle-pour_bottle) are precompiled binaries. This way people don't need to compile from source every time.

> Bottles for homebrew/core formulae are created by [Brew Test Bot](https://docs.brew.sh/Brew-Test-Bot) when a pull request is submitted. If the formula builds successfully on each supported platform and a maintainer approves the change, [Brew Test Bot](https://docs.brew.sh/Brew-Test-Bot) updates its bottle do block and uploads each bottle to GitHub Packages.

```ruby
  ...
  # IMPORTANT: These are automatically generated, you DO NOT need to add these manually, I'm adding them here as an example
  bottle do
    sha256 cellar: :any_skip_relocation, arm64_ventura:  "40434b61e99cf9114a3715851d01c09edaa94b814f89864d57a18d00a8e0c4e9"
    sha256 cellar: :any_skip_relocation, arm64_monterey: "edd6dcf9d627746a910d324422085eb4b06cdab654789a03b37133cd4868633c"
    sha256 cellar: :any_skip_relocation, arm64_big_sur:  "d9568107514168afc41e73bd3fd0fc45a6a9891a289857831f8ee027fb339676"
    sha256 cellar: :any_skip_relocation, ventura:        "d7289b5efca029aaa95328319ccf1d8a4813c7828f366314e569993eeeaf0003"
    sha256 cellar: :any_skip_relocation, monterey:       "ba58e1eb3398c725207ce9d6251d29b549cde32644c3d622cd286b86c7896576"
    sha256 cellar: :any_skip_relocation, big_sur:        "3e2431a6316b8f0ffa4db75758fcdd9dea162fdfb3dbff56f5e405bcbea4fedc"
    sha256 cellar: :any_skip_relocation, x86_64_linux:   "925113b4967ed9d3da78cd12745b1282198694a7f8c11d75b8c41451f8eff4b5"
  end
  ...
```

### Livecheck

[Brew livecheck](https://docs.brew.sh/Brew-Livecheck) uses strategies to find the newest version of a formula or cask’s software by checking upstream. The strategy used below checks for all `aptos-cli-v<SEMVER>` tags for `aptos-core`. The regex ensures that releases for other, non-CLI builds are not factored into livecheck. 

Livecheck is run on a schedule with BrewTestBot and will update the bottles automatically on a schedule to ensure they're up to date. For more info on how BrewTestBot and brew livecheck works, please see [this discussion link](https://github.com/Homebrew/discussions/discussions/3083).

```ruby
...
  # This livecheck scans the releases folder and looks for all releases
  # with matching regex of href="<URL>/tag/aptos-cli-v<SEMVER>". This
  # is done to automatically check for new release versions of the CLI.
  livecheck do
    url :stable
    regex(/^aptos-cli[._-]v?(\d+(?:\.\d+)+)$/i)
  end
...
```

To run livecheck for testing I reccommend running

```bash
brew livecheck --debug aptos
```

### Depends On and Installation

- `depends_on` is for specifying other [homebrew formulas as dependencies](https://docs.brew.sh/Formula-Cookbook#specifying-other-formulae-as-dependencies)
- Currently we use v1.64 of rust, as specified in the Cargo.toml of the project. If we were to use the latest stable build of rust 
going forward, we would modify the formula slightly. See the comments below for more details.


```ruby
  # Installs listed homebrew dependencies before aptos installation
  # Dependencies needed: https://aptos.dev/cli-tools/build-aptos-cli
  # See scripts/dev_setup.sh in aptos-core for more info
  depends_on "cmake" => :build
  depends_on "rustup-init" => :build
  uses_from_macos "llvm" => :build

  on_linux do
    depends_on "pkg-config" => :build
    # Might need to use "openssl@1.1", let's see https://docs.rs/openssl/latest/openssl/#automatic
    depends_on "openssl@3"
    depends_on "systemd"
  end

  # Currently must compile with the same rustc version specified in the
  # root Cargo.toml file of aptos-core (currently it is pegged to rust 
  # v1.64). In the future if it becomes compatible with the latest rust
  # toolchain, we can remove the use of rustup-init, replacing it with a 
  # depends_on "rust" => :build
  # above and build the binary without rustup as a dependency
  def install
    system "#{Formula["rustup-init"].bin}/rustup-init",
      "-qy", "--no-modify-path", "--default-toolchain", "1.64"
    ENV.prepend_path "PATH", HOMEBREW_CACHE/"cargo_cache/bin"
    system "cargo", "install", *std_cargo_args(path: "crates/aptos")
    bin.install "target/release/aptos"
  end
```

### Tests

To run tests simply run 

```bash
brew test aptos
```

The test that currently exists generates a new key via aptos cli and ensures that the shell output matches the filename(s) for that key.

```ruby
  ...
  test do
    assert_match(/output.pub/i, shell_output("#{bin}/aptos key generate --output-file output"))
  end
  ...
```

## FAQ

- To view other homebrew related FAQ or ask questions, please visit their [discussions board](https://github.com/orgs/Homebrew/discussions).
- For similar Rust related build examples I reccommend checking out
  - [`rustfmt.rb`](https://github.com/Homebrew/homebrew-core/blob/master/Formula/rustfmt.rb)
  - [`solana.rb`](https://github.com/Homebrew/homebrew-core/blob/master/Formula/solana.rb)
- Other guides
  - [Homebrew formula cookbook](https://docs.brew.sh/Formula-Cookbook)
  - [Creating and Running Your Own Homebrew Tap - Rust Runbook](https://publishing-project.rivendellweb.net/creating-and-running-your-own-homebrew-tap/)