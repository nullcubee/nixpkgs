# Pelican Panel {#module-pelican-panel}

Pelican Panel is a game server panel forked from Pterodactyl which manages Docker containers on multiple machines, using their control plane Wings.
Wings is required to use it and vice versa. To set it up, see the docs for `services.pelican-wings`.

You should probably read the [official docs](https://pelican.dev/docs/panel/getting-started) before continuing.

Note that the web installer is disabled by default. This means that you have to create the first admin user yourself. This can be done by running

```sh
pelican-artisan p:user:make --admin
```

All changes to `.env` will be stored in `/var/lib/pelican`, but it is recommended to configure everything through Nix where possible;
make changes to `services.pelican-panel.environment` or the `secretEnvironmentFile` is recommended.

The `secretEnvironmentFile` option has to be set to the absolute path of a file that exists. Please use a proper secret management scheme to provide it.

## Using traefik as a reverse proxy {#module-pelican-panel-traefik}

By default, the panel runs using Caddy on an insecure port (`services.pelican-panel.port`). Caddy will not listen on ports 80 and 443.
This is so you can use any reverse proxy, even another webserver like nginx.

The package has support for using traefik as a reverse proxy. See the docs for `services.traefik` on how to enable it.
For example, this could look like this:

```nix
# configuration.nix
{ pkgs, ... }:
{
  services = {
    pelican-wings = {
      # See `services.pelican-wings`
      # ...

      configuration.remote = "https://panel.your.domain";
    };

    pelican-panel = {
      enable = true;

      enableTraefik = true;
      domain = "panel.your.domain";

      # in production, use a secrets manager like agenix or sops-nix
      # secrets saved like this will be world-readable in the store
      secretEnvironmentFile = pkgs.writeText "secrets.env" ''
        APP_KEY="my-super-secret-app-key"
      '';
    };

    traefik = {
      enable = true;

      staticConfigOptions = {
        entryPoints = {
          web = {
            address = ":80";

            # Redirect http to https
            http.redirections.entryPoint = {
              to = "websecure";
              scheme = "https";
            };
          };

          websecure = {
            address = ":443";
          };
        };

        certificatesResolvers.letsencrypt.acme = {
          email = "you@your.domain";
          storage = "/var/lib/traefik/acme.json";
          tlsChallenge = { };
        };
      };
    };
  };
}
```

