# Nix Overriding go package sources

## Context

I was trying to set up my backup strategy in my Proton Drive using [restic](https://restic.net/) and rclone on NixOS, specifically using the Home-Manager modules.
One problem I encountered was that rclone wasn't working properly with Proton Drive at the time, but fortunately someone had already worked on a fix for this issue in a [fork of rclone](https://github.com/coderFrankenstain/rclone/tree/fix-protondrive-upload) (thank you btw).


## Overriding rclone in Nixpkgs

I've never really built packages with Nix before, just playing around with overrides here and there. So I decided to give it a try and override the rclone package in Nixpkgs to use this fork instead.

### Simple `overrideAttrs` approach

I tried the simple `overrideAttrs` approach:

```nix
{
  pkgs,
  ...
}: let
  rclone-protondrive-fix = pkgs.rclone.overrideAttrs (old: {
    src = pkgs.fetchFromGitHub {
      owner = "coderFrankenstain";
      repo = "rclone";
      rev = "fix-protondrive-upload";
      hash = "sha256-AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=";
    };
  });
in {
  programs.rclone = {
    package = rclone-protondrive-fix;
  };
}
```

With replacing the hash with the correct one of course. However, this didn't work:
```
> source root is source
> Running phase: patchPhase
> Running phase: updateAutotoolsGnuConfigScriptsPhase
> Running phase: configurePhase
> Running phase: buildPhase
> Building subPackage ./.
> go: inconsistent vendoring in /build/source:
>         github.com/henrybear327/Proton-API-Bridge@v1.0.0: is replaced in go.mod, but not marked as replaced in vendor/modules.txt
>       github.com/henrybear327/go-proton-api@v1.0.0: is replaced in go.mod, but not marked as replaced in vendor/modules.txt
>
>   To ignore the vendor directory, use -mod=readonly or -mod=mod.
>  To sync the vendor directory, run:
>              go mod vendor
```

### Using `-mod=mod` or `-mod=readonly`

So I tried to do as indicated, and override the `buildPhase` to use `-mod=mod` or `-mod=readonly`, but that didn't work either:

```nix
pkgs.rclone.overrideAttrs (old: {
  # [...]
  ldFlags = [ "-mod=mod" ]; # or "-mod=readonly"
})
```

This resulted in the same error as above.

### Final working solution: regenerating the vendor directory

After some research, I found out that the issue was that the `vendor` directory in the forked repository was not in sync with the `go.mod` file.
So the solution was to regenerate the `vendorHash` and override its value in the package definition:
1. Clone the forked repository locally. `git clone https://github.com/coderFrankenstain/rclone.git && cd rclone && git checkout fix-protondrive-upload`.
2. Enter a Go-capable shell and sync vendors: `nix shell nixpkgs#go -c bash`, then `go mod tidy && go mod vendor'`.
3. Calculate the new `vendorHash` using Nix: `nix hash path vendor`.
4. Finally, override the `vendorHash` in the Nix expression:

```nix
{
  pkgs,
  ...
}: let
  rclone-protondrive-fix = pkgs.rclone.overrideAttrs (old: {
    src = pkgs.fetchFromGitHub {
      # [...]
    };
    vendorHash = "new hash value here";
  });
```

With this, the package built successfully.

---

P.S.: When editing the `ldflags`, I wondered if it completedly replaced the existing flags or appended to them. I came accross the `lib.mkMerge` function, used in the nixpkgs source, but I don't know what difference it does (since I don't use it in my NixOS config). I will probably investigate this later.