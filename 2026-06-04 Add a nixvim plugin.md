# Add a Nixvim plugin

## The official documentation

In the [official Nixvim documentation](https://nix-community.github.io/nixvim/), there's a section on [how to add a plugin](https://nix-community.github.io/nixvim/user-guide/faq.html#how-do-i-use-a-plugin-not-implemented-in-nixvim). It even mentions [how to add a plugin which is not available in the Nixpkgs repository](https://nix-community.github.io/nixvim/user-guide/faq.html#how-do-i-use-a-plugin-not-packaged-in-nixpkgs):

### Use a plugin not yet implemented in Nixvim

Copy the module expression formatted like this into a file:

```nix
{ lib, ... }:
lib.nixvim.plugins.mkNeovimPlugin {
  # ...
}
```

Import it into your Nixvim configuration and configure it:

```nix
{
  # Remove this `programs.nixvim` wrapper for standalone configurations
  programs.nixvim = {
    # You could also substitute the filename with the module expression
    imports = [ ./my-plugin.nix ];

    plugins.my-plugin.enable = true;
  };
}
```

### Use a plugin not packaged in Nixpkgs

This is straightforward too, you can add the following to extraPlugins for a plugin hosted on GitHub:

```nix
{
  extraPlugins = [(pkgs.vimUtils.buildVimPlugin {
    name = "my-plugin";
    src = pkgs.fetchFromGitHub {
        owner = "<owner>";
        repo = "<repo>";
        rev = "<commit hash>";
        hash = "<nix NAR hash>";
    };
  })];
}
```

The [nixpkgs manual](https://nixos.org/manual/nixpkgs/stable/#managing-plugins-with-vim-packages) has more information on this.

## How to do it in practice

I wanted to add the [dbsession.nvim](https://github.com/nvimdev/dbsession.nvim) plugin, to test it in comparison with [persistence.nvim](https://github.com/folke/persistence.nvim).

> Note: I finally went with persistence.nvim, but the experience of adding a plugin not available in Nixpkgs was interesting.

I first had to add an overlay on Nixpkgs to add the plugin as a package. I added it in the `flake.nix`:

```nix
{
  inputs = {...};

  outputs = {...}: let
    dbsession-overlay = (final: prev: {
      vimPlugins = prev.vimPlugins.extend (final': prev': {
        dbsession-nvim = prev.vimUtils.buildVimPlugin {
          name = "dbsession.nvim";
          src = prev.fetchFromGitHub {
            owner = "nvimdev";
            repo = "dbsession.nvim";
            rev = "...";
            hash = "...";
        };
      };
    });
  }
  );
    system = "x86_64-linux";
    pkgs = nixpkgs.legacyPackages.${system}.extend dbsession-overlay;
  in {
    packages.${system}.default = pkgs.callPackage nixvim.legacyPackages.${system}.makeNixvimWithModule {
      module = {
        imports = [
          ./modules

          # ...
        ];
      };
    };
  };
}
```

Then in my `./modules` directory, I followed the Nixvim structure and added a dbsession directory (and a `default.nix` importing this directory), in which I added a `default.nix` file with the following content:

```nix
{
  lib,
  pkgs,
  ...
}: let
  inherit (lib.nixvim) defaultNullOpts literalLua;
in
lib.nixvim.plugins.mkNeovimPlugin {
  name = "dbsession";
  package = "dbsession-nvim";
  description = "a simple small powerful session for neovim";

  maintainers = [];

  settingsOptions = {
    dir = defaultNullOpts.mkStr (literalLua "vim.fn.expand(vim.fn.stdpath('state') .. '/nvim/session/')") ''
      the session store dir
    '';
    auto_save_on_exit = defaultNullOpts.mkBool false ''
      auto save session when quit neovim
    '';
  };
}
```

---

Finally, in my config I could enable the plugin:

```nix
{
  plugins.dbsession = {
    enable = true;
    settings.auto_save_on_exit = true;
  };
}
```