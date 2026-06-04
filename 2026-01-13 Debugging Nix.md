# Debugging Nix

## The Buginning (Context)

Like sometimes, I updated the flakes of my NixOS system, and like often, something broke. This time, it was my Hyprland configuration. I checked it and saw the config file was now alphabetically sorted, which defined the `$mod = Super` at the bottom, making it undefined for the bindings above it.

Thus started a search as to why it was now sorted (it's been weeks since I last updated, I use Home Manager and the Hyprland module from the official repo, so I had no idea where the change came from).

### 1st suspect: NixOS

I looked at the [Unstable NixOS release notes](https://nixos.org/manual/nixos/unstable/release-notes) if there was any mention of Hyprland, but there was none.

I looked at the [hyprland module in the NixOS repo](https://github.com/NixOS/nixpkgs/blob/ffbc9f8cbaacfb331b6017d5a5abb21a492c9a38/pkgs/by-name/hy/hyprland/package.nix) but the last commits were about UWSM (which might cause other issues, I hope not) so NixOS should be innocent.

### 2nd suspect: Hyprland's flake

I checked the [Hyprland flake](https://github.com/hyprwm/Hyprland/blob/8d03fcc8d76245be013254ea30fbe534f680dc9f/flake.nix) but again, no luck. Recent commits on it were about the CI and nothing about configuration changes.

### 3rd suspect: Home Manager

I quickly looked at the [Home Manager Hyprland module](https://github.com/nix-community/home-manager/blob/360620ec9d5ffc186b5b1fca4c5f5b2e1634a5cf/modules/services/window-managers/hyprland.nix) but again, the recent changes were about recompiling when xwayland, nothing about configuration. It was getting late so I called it a ~~day~~ night and went to sleep on it.

I had an idea this night, which should have come to me earlier: what about the way Home Manager generates the config file. Is it a function from another file that might have changed? I quickly checked it in the morning, and bingo: [the culprit](https://github.com/nix-community/home-manager/commit/d4e4d5cfa30455f50d034f24aa9bc0ab05388fcd).

What I don't get is that I didn't re-define the importantFields option, so its default value (`["$" "bezier" "name" "output"]`) should be applied, and `$mod` should be at the top. That is still a mystery to solve.

## Skilling up Nix debugging

So far when something wasn't working in Nix, it would be try-and-error, reading the docs, checking the built derivations to see why my configuration wasn't good, etc.

I decided to skill up my Nix debugging skills. It's been a while (> 3 years) since I started using Nix as my main package manager and OS config tool, so more than time to get better at it.

I guessed the go-to tool would be `nix repl`, so I started exploring with it:
- Like any good REPL, it indicates how to get help: `:?`.
- You can execute Nix expressions directly in it.
- There's a project, [nix-inspect](https://github.com/bluskript/nix-inspect) that provide a TUI instead of the REPL to explore Nix expressions. I tried it quickly (`nix run github:bluskript/nix-inspect -- -p .` in my NixOS config repo) and it looks nice, but let's confine myself to the REPL for now.
- You can load files with `:l <file>`, and flakes with `:lf <flake-uri>`. For example, being in my NixOS config repo, I simply typed `:lf .`. Nix then tells you how many variables were loaded. You can then access the variables defined in your flake. For example, I could type `outputs.nixosConfigurations.<system-name>.config` to get my NixOS configuration or any sub-variable from it.
- So once I loaded my flake, I tried to load my Home Manager config: `hm = inputs.home-manager.lib.homeManagerConfiguration; hm { pkgs = inputs.nixpkgs.legacyPackages."x86_64-linux"; modules = [./homes/tibo/default.nix]; }`. But no luck here: I couldn't load my Home Manager config since it was dependent on the NixOS system config (`home.stateVersion = osConfig.system.stateVersion;` idk if it's bad practice).
- You can also use `:b <expression>` to build an expression. For example, I could do `:b inputs.nixpkgs.legacyPackages."x86_64-linux".hello` to build the hello package and see the output derivation.

The next goal will be to be able to load my Home Manager config in the REPL, so I can explore how the Hyprland config file is generated.

## Next day

I couldn't load the Home Manager config but I didn't need to:
- I should have searched in `hmLib = inputs.home-manager.lib.hm;` instead to allow me access the Home Manager functions (like the Hyprland config's generator `hmLib.generators.toHyprconf`). Then, by loading my Hyprland config (`:lf .; hmConf = nixosConfigurations.<system-name>.config.home-manager.users.<user>;`) I could call the generator directly with `hmLib.generators.toHyprconf { attrs = wayland.windowManager.hyprland.settings; }` and see how the config file was generated. The `$mod` was indeed at the top.

## Conclusion

I'm dumb. Like really. I don't know why but I redefined the `importantFields` option sooner in my config, its value was `[ "origin"]` so of course `$mod` would be at the bottom. When checking with `nix repl`, I saw the generated config file had `$mod` at the top because I didn't load the total config, only the bindings, and the `importantFields` option was redefined in the general settings.nix file...

It allowed me to learn how to use `nix repl` better, so not a total loss.

## References

- [Nix REPL Home Manager](https://gist.github.com/573/c06ccea910abcc8f9255cb8f3ace3e86)
- [Nix REPL Manual](https://nixos.org/manual/nix/latest/command-ref/new-cli/nix3-repl.html)