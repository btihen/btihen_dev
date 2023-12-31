---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Phoenix 1.6 Passwordless (2F Auth)"
subtitle: "Using Tokens instead of Passwords"
summary: "Create a modern Phoenix SPA with tremendous flexibility"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "Passwordless", "Tokens"]
categories: ["Code", "Elixir Language", "Phoenix Framework"]
date: 2022-02-27T01:01:53+02:00
lastmod: 2022-02-27T01:01:53+02:00
featured: false
draft: true

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---



## Resources

### Confirmation Cache
- https://thoughtbot.com/blog/make-phoenix-even-faster-with-a-genserver-backed-key-value-store

### Passwordless

- (Phoenix Tokens) - https://hexdocs.pm/phoenix/Phoenix.Token.html#verify/4
- (nwjlyons/phx_gen_auth_passwordless) - https://github.com/nwjlyons/phx_gen_auth_passwordless/tree/master/priv/templates/phx.gen.auth.passwordless

- Rick Syllivan, (7 Jun 2018) - https://ricksullivan.net/posts/passwordless-login
- Ricardo García Vega, (Jun 9, 2018) - https://bigardone.dev/blog/2018/06/09/elixir-and-phoenix-basic-passwordless-and-databaseless-authentication-pt-1
- Ricardo García Vega, (Jun 20, 2018) - https://bigardone.dev/blog/2018/06/20/elixir-and-phoenix-basic-passwordless-and-databaseless-authentication-pt-2
- Ricardo García Vega, https://github.com/bigardone/passwordless-auth
- Dennis Reimann, (09 Jan 2017) - https://dennisreimann.de/articles/phoenix-passwordless-authentication-magic-link.html


### Authentication

- DORIAN ROUSSEL, (30 AUG 2021) - https://blog.kalvad.com/handle-authentication-with-phoenix-framework/
- Nithin Bekal, (30 Jun 2015) - https://nithinbekal.com/posts/phoenix-authentication/
- https://powauth.com/
- Cameron Carlyle, (Feb 2021) - https://levelup.gitconnected.com/a-deep-dive-into-authentication-in-elixir-phoenix-with-phx-gen-auth-9686afecf8bd
- , () - https://fullstackphoenix.com/tutorials/combining-authentication-solutions-with-guardian-and-phx-gen-auth
- Scott Hamilton, (5 May 2019) - https://shamil614.github.io/programming/2019/05/05/authentication_and_authorization_with_graphql.html
- Kuba Kowalczykowski, (9.08.2021) - https://prograils.com/elixir-phoenix-authorization-authentication

- James More, KnowThen, (11.02.2020) - Easy Authentication in Elixir & Phoenix with the pow & pow_assent libraries, https://www.youtube.com/watch?v=hnD0Z0LGMIk
- James More, KnowThen, (27.02.2020) - Authorization in Phoenix web applications using Role Based Access Control (RBAC), https://www.youtube.com/watch?v=6TlcVk-1Tpc


### Authorization (including LiveView)

- José Valim, (26 Mar 2020), An upcoming authentication solution for Phoenix, https://dashbit.co/blog/a-new-authentication-solution-for-phoenix

- James More, KnowThen, (11.02.2020) - Easy Authentication in Elixir & Phoenix with the pow & pow_assent libraries, https://www.youtube.com/watch?v=hnD0Z0LGMIk
- James More, KnowThen, (27.02.2020) - Authorization in Phoenix web applications using Role Based Access Control (RBAC), https://www.youtube.com/watch?v=6TlcVk-1Tpc

- Oliver Andrich, (10 Jan 2021) - https://dev.to/oliverandrich/how-to-connect-pow-and-live-view-in-your-phoenix-project-1ga1
- MICHAŁ BUSZKIEWICZ, (Nov 2020) - https://curiosum.com/blog/elixir-phoenix-liveview-messenger-part-3#authenticate-with-pow
- João Gilberto Balsini Moura, (October 16, 2020) - https://www.leanpanda.com/blog/authentication-and-authorisation-in-phoenix-liveview/
- Hari Roshan, (6 Oct 2020) - https://medium.com/@hariroshanmail/authentication-and-authorization-in-phoenix-live-view-5ad677a03ef
- https://www.keycloak.org/docs/latest/authorization_services/index.html

- https://auth0.com/developers/hub/code-samples/api/phoenix-elixir/basic-role-based-access-control


### Two Factor Authentication

- Cam Carter, (18 March 2019) - https://teamgaslight.com/blog/two-factor-authentication-in-elixir-and-phoenix

- https://hexdocs.pm/one_time_pass_ecto/OneTimePassEcto.html
- Marlus Saraiva, (1 Aug 2020) - Introducing Nimble TOTP, https://dashbit.co/blog/introducing-nimble-totp
- https://github.com/dashbitco/nimble_totp/
- https://hexdocs.pm/nimble_totp/NimbleTOTP.html
- (QR Codes for NibleTOTP Secret) https://github.com/SiliconJungles/eqrcode

- https://github.com/yuce/pot (Google Auth Compatible)
- https://github.com/riverrun/one_time_pass_ecto
