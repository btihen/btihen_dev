---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Phoenix / RabbitMQ - Integration with Broadway"
subtitle: "Sending and receiving messages with RabbitMQ"
# Summary for listings and search engines
summary: "Solutions to stumbling blocks when integrating with RabbitMQ within Phoenix"
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "RabbitMQ", "Broadway"]
categories: ["Code", "Elixir Language", "Phoenix Framework", "RabbitMQ"]
date: 2024-07-28T01:01:53+02:00
lastmod: 2024-07-28T01:01:53+02:00
featured: true
draft: false

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


## Introduction

## RabbitMQ Setup

```
# AVK Service and Friends

1. have running GREM (using docker)
```bash
# starts up our entire ecosystem
docker compose up -d
# starts up GREM processes
bin/start
```

2. enable avk-service within GREM
```ruby

bin/rails c

# feom .env.development
# of course update / change as needed
# export AVK_SERVICE_CLIENT_ID="aBjduMM1Eh"
# export AVK_SERVICE_CLIENT_SECRET="979z8oNieHu6Dg5D9Ps5yywNxpKsLW30"

client_id = 'aBjduMM1Eh'
encrypted_client_secret = User.new(password: '979z8oNieHu6Dg5D9Ps5yywNxpKsLW30').encrypted_password

name = 'avk_service'
scopes = ['accounting', 'avk_service', 'supplier']
puts encrypted_client_secret

service = ExternalService.create(name:, client_id:, scopes:, encrypted_client_secret:)
# #<ExternalService:0x000000010a5117c8
#  id: 10,
#  name: "avk_service",
#  client_id: "aBjduMM1Eh",
#  encrypted_client_secret: "$2a$11$QyWLpyKozLFoY9oGBSvrbeK5kI1GhIiwmvjFENUPYSnMg2nWiidki",
#  scopes: ["accounting", "avk_service", "supplier"],
#  created_at: 2024-07-23 16:00:17.054981 +0200,
#  updated_at: 2024-07-23 16:00:17.054981 +0200,
#  objektkategorie_cds: [],
#  notification_kategorie_cd: nil,
#  active: true>

# or directly in DB
bin/rails db
INSERT INTO external_services(name,client_id,encrypted_client_secret,scopes,created_at,updated_at) VALUES('avk_service', 'aBjduMM1Eh', '$2b$12$K.aDGjHcbJgoeQpXrKdHcuHgAnO/AMpYgN.eEkicvuSH93LlEu/Ja', '["accounting", "avk_service", "supplier"]', NOW(), NOW());
# INSERT 0 1
\q

# or UPDATE
# service = ExternalService.find_by(name:, client_id:)
# service.encrypted_client_secret = encrypted_client_secret
# service.save
```

3. setup graphql-backend (locally) - uses port 4000

Be sure to point to the same DB here (with the same values):
```bash
# config/dev_adapter_config.exs
config :garaio_rem_graphql, GaraioRemGraphql.Repo,
  adapter: Ecto.Adapters.Postgres,
  username: {:system, "DATABASE_USER", "rem2"},
  password: {:system, "DATABASE_PW", "xxxxx"},
  database: {:system, "DATABASE", "omit"},
  hostname: "localhost",
  port: 5432,
  pool_size: 10,
  plan_cache_mode: "force_custom_plan",
  types: GaraioRemGraphql.PostgresTypes
```
as here in GREM:
```
development: &development
  adapter: postgis
  username: rem2
  password: xxxx
  database: omit
  encoding: unicode
  host: localhost
  port: 5432
  gssencmode: disable # enable if Puma Server crashes with docker
```

now start graphql and test:
```elixir
iex -S mix phx.server
http://localhost:4000/api/graphiql

# user Auth (deprecated)
query {
        authenticate(username: "admgar", password: "rem2InFuture!") {
          accessToken
        }
      }
# {
#   "data": {
#     "authenticate": {
#       "accessToken": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJHYXJhaW9SZW1HcmFwaHFsIiwiZXhwIjoxNzIxNzQ3NjA4LCJpYXQiOjE3MjE3NDQwMDgsImlzcyI6IkdhcmFpb1JlbUdyYXBocWwiLCJqdGkiOiI0YzM2MjlkYS1iODMzLTQzOWYtOTFlNi1iNDMwZTdkMDRjNTEiLCJuYmYiOjE3MjE3NDQwMDcsInN1YiI6IntcImlkXCI6NixcInR5cGVcIjpcInVzZXJcIn0iLCJ0eXAiOiJhY2Nlc3MifQ.Dpv_AkkAOkLg7jkljKJlVp1_zUSFf_eapQ9DjNjjJT0UZgPSDZ0g78e1t5JrtmRtZxHZMiVaRFKWXB6hnAN-ew"
#     }
#   }
# }

# deprecated
query {
       authenticateService(clientId: "aBjduMM1Eh", clientSecret: "979z8oNieHu6Dg5D9Ps5yywNxpKsLW30") {
          accessToken
        }
}
# response
{
  "data": {
    "authenticateService": {
      "accessToken": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJHYXJhaW9SZW1HcmFwaHFsIiwiZXhwIjoxNzI0MzU3ODg2LCJpYXQiOjE3MjE3NjU4ODYsImlzcyI6IkdhcmFpb1JlbUdyYXBocWwiLCJqdGkiOiI2MjNmNmE4MS1mMjI1LTQ1NTAtYmFlZC0xZmFjOWI0ZjE3NDciLCJuYmYiOjE3MjE3NjU4ODUsInN1YiI6IntcImlkXCI6MTEsXCJ0eXBlXCI6XCJzZXJ2aWNlXCJ9IiwidHlwIjoiYWNjZXNzIn0.T0Qq8bHRs5VsXSUI7Yj4GePuEWUu83QwvCw9JrgymudK2j_rE-hQl56Rhbqf1slIaRBJ393fq4NBrJfDlwrK7A"
    }
  }
}

# if you get this - check that you are using th same db everywhere!
# {
#   "data": {
#     "authenticateService": null
#   },
#   "errors": [
#     {
#       "message": "Incorrect client id or client secret",
#       "path": [
#         "authenticateService"
#       ],
#       "extensions": {
#         "code": "invalid_grant"
#       },
#       "locations": [
#         {
#           "line": 2,
#           "column": 8
#         }
#       ]
#     }
#   ]
# }

# test that you can login with the avk_service
curl --request POST \
  --url 'http://localhost:4000/api/graphql/authenticate/token' \
  --header 'content-type: application/x-www-form-urlencoded' \
  --data grant_type=client_credentials \
  --data client_id="aBjduMM1Eh" \
  --data client_secret="979z8oNieHu6Dg5D9Ps5yywNxpKsLW30"

# response
{"access_token":"eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJHYXJhaW9SZW1HcmFwaHFsIiwiZXhwIjoxNzI0MzU4MDI1LCJpYXQiOjE3MjE3NjYwMjUsImlzcyI6IkdhcmFpb1JlbUdyYXBocWwiLCJqdGkiOiI5MDEyZDc0MC03MWE3LTQyM2MtOTI3ZS0yMzk4YzVhMmIxOGIiLCJuYmYiOjE3MjE3NjYwMjQsInN1YiI6IntcImlkXCI6MTEsXCJ0eXBlXCI6XCJzZXJ2aWNlXCJ9IiwidHlwIjoiYWNjZXNzIn0.hm1Awy8jNXjyazd4j1w9HLNzmG9r2XiJhZxmcNzuycA24bq2no71nmSHUDlC7O0MIQhVj427V4YfMecAsKhT9Q"}%
# if you get this - check that you are using th same db everywhere!
# {"error":"invalid_grant","error_description":"Incorrect client id or client secret"}%
```

4. Setup RabbitMQ
```bash
# open Rabbit MQ 'Overview' Tab
https://mbus.garaio-rem.ch/#/

# open 'Export definitions'
# click on 'Download broker definitions'
```

now open:
```bash
# open Rabbit MQ 'Overview' Tab
http://localhost:15672/#/

# open 'Import definitions'
# click on 'Choose File'
# click on 'Upload broker definitions'
```

NOTE: NOW you will need to logout and login again with the credentials from the 'official' mbus server!


5. setup avk-service (locally) - change port to 4040 (so as not to conflict with graphql backend)
```bash
export PHX_PORT=4040

iex -S mix phx.server
```

# GREM rabbitMQ Setup

Now that RabbitMQ is setup check/configure the GREM setup:
```yml
# config/rabbitmq.yml
development: &development
  enabled: true
  host: localhost
  vhost: /
  port: 5672
  ssl: false
  username: admin
  password: same_as_mbus_now (if mbus is imported)
  exchange_name: grem_publish
  verify_peer: false
```

# setup AVK-Service (User sicht)

Set Tenant (localhost) with:

name: `localhost`
graphql url: `localhost:4000/api`


URL:
https://avk-service.fly.dev/invoices/5498f54a-678c-4507-8809-67f569068e0e.pdf

## send message 'Invoicing.Invoice.Created'

## response messages 'Invoicing.Invoice.Accepted'

```ruby
# app/amqp/handlers/avk_invoice_message_handler.rb

    def success_result
      Result.success(
        business_event: BusinessEvent.new_for_message(avk_verantwortlicher, @command.class.name, @command.command_object)
      )
    end

    def send_invoice_accepted
      send_reply(
        'Invoicing.Invoice.Accepted',
        {
          externalReference: context_object.externe_rechnungs_nr,
          reference: context_object.referenz,
          orderReference: kreditorenauftrag&.referenz,
          externalOrderReference: kreditorenauftrag&.externe_auftrags_nr
        }
      )
    end


```

```json
headers:
{
  "app_id": "grem_derham-test",
  "NewRelicID": "VgIGVV9WGwICVVBXBQQ=",
  "unit_category_code": "n/a",
  "NewRelicTransaction": "PxQABFEHAFYBV1EHAAkEVgADFB8EBw8RVU4aVV0PUApWVApRVFYLUVRSBUNKQQ9VBlwAW1EJFTs="
}

Message:
{
  "eventType": "Invoicing.Invoice.Accepted",
  "data": {
    "reference": null,
    "externalReference": "derham-test-025"
  }
}

{"externalReference": "301186-17544"}
```


## response messages 'Invoicing.Invoice.Rejected'
```ruby
# app/amqp/handlers/avk_invoice_message_handler.rb
    def error_result
      ErrorResponse
        .new({})
        .tap { _1.add_handler_errors(errors) }
        .then { Result.error(_1.reasons_hash) }
    end

    def send_invoice_rejected
      send_reply(
        'Invoicing.Invoice.Rejected',
        {
          externalReference: @communication.request_field('externalReference'),
          reference: @communication.request_field('reference'),
          reasons: build_reject_reasons
        }
      )
    end
```

```json
headers:
{
  "app_id": "grem_derham-test",
  "NewRelicID": "VgIGVV9WGwICVVBXBQQ=",
  "unit_category_code": "n/a",
  "NewRelicTransaction": "PxRVVFFTDgVWVlkBUlNVB1ZSFB8EBw8RVU4aAA0PBAQFAwtZUgRRAAUEVENKQQ9VBlwAW1EJFTs="
}

message:
{
  "eventType": "Invoicing.Invoice.Rejected",
  "data": {
    "reasons": [
      {"reason": "Auf das Konto 1111.465000 darf nicht ohne MWST-Code gebucht werden", "attribute": "taxCode", "lineNumber": 1},
      {"reason": "Dieses Konto muss mit einem Lebenslauf-Objekt bebucht werden", "attribute": "masterdataReference", "lineNumber": 1},
      {"reason": "Dieses Konto verlangt eine Objekt Referenz und nicht die einer LG oder eines Hauses", "attribute": "masterdataReference", "lineNumber": 1},
      {"reason": "Buchungstext obligatorisch", "attribute": "bookingText", "lineNumber": 1},
      {"reason": "Angabe von LG / Haus / Objekt ist bei diesem Konto obligatorisch", "attribute": null, "lineNumber": 1},
      {"reason": "ist ungültig", "attribute": "accountNumber", "lineNumber": 2},
      {"reason": "ist ungültig", "attribute": "accountNumber", "lineNumber": 3}],
    "externalReference": "derham-test-027"
  }
}

{
  "reasons": [{"reason": "est non valable", "attribute": "iban", "lineNumber": null}],
  "externalReference": "1709"
}
```

## Base Config

```elixir
```elixir
import Config

if System.get_env("PHX_SERVER") do
  config :avk_service, AvkServiceWeb.Endpoint, server: true
end

if config_env() != :test do
  # RabbitMQ Config
  config :avk_service, :rabbitmq,
    connection_params: [
      host:
        System.get_env("RABBITMQ_HOST") ||
          raise("Missing env variable: RABBITMQ_HOST"),
      virtual_host:
        System.get_env("RABBITMQ_VHOST") ||
          raise("Missing env variable: RABBITMQ_VHOST"),
      port:
        String.to_integer(System.get_env("RABBITMQ_PORT")) ||
          raise("Missing env variable: RABBITMQ_PORT"),
      username:
        System.get_env("RABBITMQ_USER") ||
          raise("Missing env variable: RABBITMQ_USER"),
      password:
        System.get_env("RABBITMQ_PASSWORD") ||
          raise("Missing env variable: RABBITMQ_PASSWORD")
    ]
end
```

## Send to a Queue

```elixir
# Usage
# data = %{greeting: "Hello, Broadway!"}
# AvkService.Amqp.QueuePublisher.send_message(data, "grem_qs06")
defmodule AvkService.Amqp.QueuePublisher do
  @moduledoc """
  Handles publishing messages to RabbitMQ.
  """

  alias AMQP.{Basic, Channel, Connection}
  alias Jason

  def send_message(message, queue \\ "avk_service") do
    config = Application.fetch_env!(:avk_service, :rabbitmq)[:connection_params]
    json_message = Jason.encode!(message)

    with {:ok, connection} <- Connection.open(config),
         {:ok, channel} <- Channel.open(connection) do
      Basic.publish(channel, "", queue, json_message)
      IO.puts("Sent message: #{json_message}")

      Channel.close(channel)
      Connection.close(connection)
      :ok
    else
      {:error, reason} ->
        IO.puts("Failed to send message: #{inspect(reason)}")
    end
  end
end
```

## Send to an Exchange (with Routing Key)


### Implementation

```elixir
defmodule AvkService.Boundary.AmqpPublisher do
  @moduledoc """
  Provides the function `publish_message/4` to publish a message on RabbitMQ
  """
  use AMQP
  require Logger

  @behaviour AvkService.Boundary.AmqpPublisherBehaviour
  def send(data, tenant_name, message_id, event_type, app_id) do
    exchange = "#{app_id}_publish"
    routing_key = event_type
    message = Jason.encode!(%{eventType: event_type, data: data})
    recipient = "grem_#{tenant_name}"

    metadata = [
      app_id: app_id,
      message_id: message_id,
      headers: [app_id: app_id, recipient: recipient]
    ]

    case publish_message(exchange, routing_key, message, metadata) do
      :ok ->
        :ok

      {:error, reason} ->
        Logger.error("ERROR: #{inspect(reason)}")
        {:error, reason}

      _ ->
        Logger.error("ERROR fetching rabbitmq config")
        {:error, :internal}
    end
  end

  defp publish_message(exchange, routing_key, message, metadata) do
    full_metadata = default_metadata() |> Keyword.merge(metadata)

    data_package = %{
      exchange: exchange,
      routing_key: routing_key,
      message: message,
      metadata: full_metadata
    }

    log_msg =
      "RabbitMQ Message Initiation: #{inspect(data_package)}"
      |> String.trim_trailing("\n")

    Logger.info(log_msg)

    with {:ok, rabbit_conn, channel} <- open_channel(),
         :ok <- Confirm.select(channel),
         :ok <- try_publish(channel, message, metadata, exchange, routing_key),
         :ok <- close_channel(rabbit_conn, channel) do
      :ok
    else
      {:error, reason} ->
        Logger.error("ERROR: #{inspect(reason)}")
        {:error, "publish error: '#{inspect(reason)}'"}

      reason ->
        {:error, "publish error: '#{inspect(reason)}'"}
    end
  end

  defp open_channel do
    with rabbitmq_config <- Application.fetch_env!(:avk_service, :rabbitmq),
         {:ok, rabbit_conn} <- Connection.open(rabbitmq_config[:connection_params]),
         {:ok, channel} <- Channel.open(rabbit_conn) do
      {:ok, rabbit_conn, channel}
    else
      {:error, :econnrefused} ->
        {:error, "Cannot connect to MBus, check RabbitMQ connection ENV variables"}

      {:error, reason} ->
        {:error, "channel error: '#{inspect(reason)}'"}

      reason ->
        {:error, "channel error: '#{inspect(reason)}'"}
    end
  end

  # Publishing to RabbitMQ is asynchronous by design. To catch a channel error
  # when publishing to an exchange that doesn't exist, we need to use Confirm
  # and explicitly handle an :exit exception.
  defp try_publish(channel, message, metadata, exchange, routing_key) do
    with :ok <- Basic.publish(channel, exchange, routing_key, message, metadata),
         true <- Confirm.wait_for_confirms_or_die(channel) do
      data_package = %{
        channel: channel,
        exchange: exchange,
        routing_key: routing_key,
        message: message,
        metadata: metadata
      }

      log_msg =
        "RabbitMQ Message Confirmed: #{inspect(data_package)}"
        |> String.trim_trailing("\n")

      Logger.info(log_msg)

      :ok
    else
      {:error, err} -> {:error, err}
      _ -> {:error, :unknown}
    end
  rescue
    err -> {:error, err}
  end

  defp close_channel(rabbit_conn, channel) do
    with :ok <- Channel.close(channel),
         :ok <- Connection.close(rabbit_conn) do
      :ok
    else
      {:error, err} -> {:error, err}
    end
  end

  defp default_metadata do
    [
      content_type: "application/json",
      timestamp: DateTime.utc_now() |> DateTime.to_unix(),
      persistent: true
    ]
  end
end
```

### Behavior
behavior allows us to mock out the rabbitmq connection for testing.

```elixir
defmodule AvkService.Boundary.AmqpPublisherBehaviour do
  @moduledoc """
  Defines the behaviour for publishing messages to RabbitMQ.
  Enables us to make the publishing of messages to for tests.
  """
  @callback send(
              message_data :: Map.t(),
              tenant_name :: String.t(),
              message_id :: String.t(),
              event_type :: String.t(),
              app_id :: String.t()
            ) :: {:ok} | {:error, String.t()}
end
```

### Test setup

```elixir
# test/support/mocks.ex
Mox.defmock(AvkService.Boundary.StorageMock, for: AvkService.Boundary.StorageBehaviour)
Mox.defmock(AvkService.Boundary.GraphQLMock, for: AvkService.Boundary.GraphQLBehaviour)

Mox.defmock(AvkService.Boundary.ExtractionApiMock,
  for: AvkService.Boundary.ExtractionApiBehaviour
)

Mox.defmock(AvkService.Boundary.AmqpPublisherMock,
  for: AvkService.Boundary.AmqpPublisherBehaviour
)
```


```elixir
import Config

# Configure your database
#
# The MIX_TEST_PARTITION environment variable can be used
# to provide built-in test partitioning in CI environment.
# Run `mix help test` for more information.
config :avk_service, AvkService.Repo,
  username: System.get_env("POSTGRES_USER", "rem2"),
  password: System.get_env("POSTGRES_PW", ""),
  hostname: System.get_env("POSTGRES_HOST", "localhost"),
  database: "avk_service_test#{System.get_env("MIX_TEST_PARTITION")}",
  pool: Ecto.Adapters.SQL.Sandbox,
  pool_size: System.schedulers_online() * 2

# We don't run a server during test. If one is required,
# you can enable the server option below.
config :avk_service, AvkServiceWeb.Endpoint,
  http: [ip: {127, 0, 0, 1}, port: 4002],
  secret_key_base: "p8wGTGKppAKj3N+r7iTdo9w0WfWRSeRMt7Bzv8wNzw0mvvHn1LtlolXPijRZJSdX",
  server: false

# DO NOT use rabbitmq consumer in tests (its not available)
# AvkService.Boundary.AmqpConsumer,
config :avk_service, :application_children, [
  AvkServiceWeb.Telemetry,
  AvkService.Repo,
  {DNSCluster, query: Application.get_env(:avk_service, :dns_cluster_query) || :ignore},
  {Phoenix.PubSub, name: AvkService.PubSub},
  # Start the Finch HTTP client for sending emails
  {Finch, name: AvkService.Finch},
  # Start a worker by calling: AvkService.Worker.start_link(arg)
  # {AvkService.Worker, arg},
  # Start to serve requests, typically the last entry
  AvkServiceWeb.Endpoint,
  # PromEx should be started after the Endpoint, to avoid unnecessary error messages
  AvkService.PromEx,
  # invoice pdf processing pipeline
  AvkService.Pipeline.PdfFilesQueue,
  {AvkService.Pipeline.BroadwayPdfToInvoice, []}
]

config :avk_service, :parashift_api,
  token: "token",
  endpoint: "https://api.parashift.io/v2",
  classification_scope: "garaio-pp-invoice-ch"

config :avk_service, :dti_api,
  app_key: "123",
  endpoint: "https://cloudextract-studio.dti.ch/extraction/api/",
  plan_id: "123"

config :tesla, adapter: Tesla.Mock

# Stub Req requests in tests
config :avk_service, graphql_req_options: [plug: {Req.Test, AvkService.Boundary.GraphQL}]

config :avk_service,
  cloud_extract_dti_api: [plug: {Req.Test, AvkService.Boundary.CloudextractDtiApi}],
  cloud_extract_parashift_api: [plug: {Req.Test, AvkService.Boundary.ParashiftApi}]

# In test we don't send emails.
config :avk_service, AvkService.Mailer, adapter: Swoosh.Adapters.Test

# Disable swoosh api client as it is only required for production adapters.
config :swoosh, :api_client, false

# Print only errors during test
config :logger, level: :none

# Initialize plugs at runtime for faster test compilation
config :phoenix, :plug_init_mode, :runtime

config :phoenix_live_view,
  # Enable helpful, but potentially expensive runtime checks
  enable_expensive_runtime_checks: true

config :junit_formatter,
  report_file: "report_file_test.xml",
  report_dir: "./ci_reports",
  print_report_file: true,
  prepend_project_name?: true

config :avk_service, :admin_basic_auth,
  username: "testadmin",
  password: "testpassword"

config :avk_service, :graphql_api,
  client_id: "test_client_id",
  client_secret: "test_client_secret"

# Boundary Modules where we cannot mock the API
config :avk_service, :boundaries, storage: AvkService.Boundary.StorageMock
config :avk_service, :boundaries, graphql: AvkService.Boundary.GraphQLMock
config :avk_service, :boundaries, extraction_api: AvkService.Boundary.ExtractionApiMock
config :avk_service, :boundaries, amqp_publisher: AvkService.Boundary.AmqpPublisherMock

# RabbitMQ Config
config :avk_service, :rabbitmq,
  connection_params: [
    host: "localhost",
    virtual_host: "/",
    port: 5672,
    username: "admin",
    password: "admin"
  ]
```

### Test Implementation

```elixir
defmodule AvkService.Pipeline.Worker.PublishInvoiceTest do
  use AvkService.DataCase, async: true
  import Mox
  import AvkService.Factory

  alias AvkService.Boundary.AmqpPublisherMock
  alias AvkService.Invoicing.Invoice
  alias AvkService.Pipeline.Worker.PublishInvoice
  alias AvkService.Repo

  describe "process/1" do
    setup do
      now = DateTime.utc_now() |> DateTime.to_date()
      due_date = Date.add(now, 30) |> Date.to_string()
      invoice_date = Date.add(now, -10) |> Date.to_string()

      storage_filepath = Ecto.UUID.generate() <> ".pdf"

      # purposely omitting external_order_reference - it should not included in the message
      invoice =
        insert(:invoice,
          iban: "CH9300762011623852957",
          external_reference: "123456789",
          storage_filepath: storage_filepath,
          accounting_reference: "1000",
          creditor_reference: "2345",
          order_reference: "34567",
          invoice_date: invoice_date,
          due_date: due_date,
          stage: "account_proposal",
          state: "OK"
        )

      insert_list(2, :invoice_line_item, invoice: invoice)
      invoice = invoice |> Repo.preload([:tenant, :line_items])

      %{invoice: invoice}
    end

    test "send expected 'Invoicing.Invoice.Created' message", %{invoice: invoice} do
      total_gross_amount =
        invoice.line_items
        |> Enum.map(& &1.booking_amount)
        |> Enum.reduce(Decimal.new("0"), &Decimal.add/2)
        |> Decimal.to_string()

      AmqpPublisherMock
      |> expect(:send, fn message_data, tenant_name, message_id, event_type, app_id ->
        assert app_id == "avk_service"
        assert event_type == "Invoicing.Invoice.Created"
        assert message_id == "invoice-#{invoice.id}"
        assert tenant_name == invoice.tenant.name

        assert message_data[:accountingReference] == invoice.accounting_reference
        assert message_data[:creditorReference] == invoice.creditor_reference
        assert message_data[:orderReference] == invoice.order_reference
        assert message_data[:totalGrossAmount] == total_gross_amount
        assert message_data[:languageCode] == invoice.language_code
        assert message_data[:iban] == invoice.iban
        assert message_data[:esrReference] == invoice.qr_reference
        assert message_data[:dueDate] == invoice.due_date
        assert message_data[:invoiceDate] == invoice.invoice_date

        assert message_data[:documentUrl] ==
                 "https://avk-service.fly.dev/invoices/" <> invoice.storage_filepath

        # Check the line_items exist with values
        assert Map.has_key?(message_data, :invoiceItems)
        invoice_items = message_data[:invoiceItems]

        # verify line items have the required fields
        keys = [:lineNumber, :taxCode, :accountNumber, :bookingAmount, :costCenterNumber]

        Enum.each(keys, fn key ->
          assert Enum.all?(invoice_items, fn item ->
                   Map.has_key?(item, key) and not is_nil(item[key])
                 end)
        end)

        # Verify that each accountNumber matches the pattern "accounting_reference.account_number"
        accounting_reference = invoice.accounting_reference

        assert Enum.all?(invoice_items, fn item ->
                 case String.split(item[:accountNumber], ".") do
                   [^accounting_reference, _account_number] -> true
                   _ -> false
                 end
               end)

        :ok
      end)
    end

    test "properly updates invoice", %{invoice: invoice} do
      AmqpPublisherMock
      |> expect(:send, fn _data, _tenant_name, _message_id, _event_type, _app_id ->
        :ok
      end)

      {:ok, published_invoice} = PublishInvoice.process(%{invoice: invoice})

      assert published_invoice.id == invoice.id
      assert published_invoice.stage == "publish_invoice"
      assert published_invoice.state == "PUBLISHED"
    end

    test "`totalGrossAmount` without line items", %{invoice: invoice} do
      hd(invoice.line_items)
      |> Ecto.Changeset.change(%{account_number: nil})
      |> Repo.update!()

      invoice =
        Repo.get!(Invoice, invoice.id)
        |> Repo.preload([:tenant, :line_items])

      total_gross_amount =
        invoice.line_items
        |> Enum.map(& &1.booking_amount)
        |> Enum.reduce(Decimal.new("0"), &Decimal.add/2)
        |> Decimal.to_string()

      AmqpPublisherMock
      |> expect(:send, fn message_data, _tenant_name, _message_id, _event_type, _api_id ->
        assert message_data[:totalGrossAmount] == total_gross_amount
        assert message_data[:invoiceItems] == []
        :ok
      end)

      {:ok, updated_invoice} = PublishInvoice.process(%{invoice: invoice})

      assert updated_invoice.stage == "publish_invoice"
      assert updated_invoice.state == "PUBLISHED"
    end

    test "absent optional data is not sent", %{invoice: invoice} do
      invoice =
        invoice
        |> Ecto.Changeset.change(%{due_date: nil})
        |> Repo.update!()

      AvkService.Boundary.AmqpPublisherMock
      |> expect(:send, fn message_data, _tenant_name, _message_id, _event_type, _app_id ->
        refute Map.has_key?(message_data, :due_date)
        :ok
      end)

      {:ok, updated_invoice} = PublishInvoice.process(%{invoice: invoice})

      assert updated_invoice.stage == "publish_invoice"
      assert updated_invoice.state == "PUBLISHED"
    end

    test "error with missing required line item data", %{invoice: invoice} do
      hd(invoice.line_items)
      |> Ecto.Changeset.change(%{booking_amount: nil})
      |> Repo.update!()

      AvkService.Boundary.AmqpPublisherMock
      |> expect(:send, fn _data, _tenant_name, _message_id, _event_type, _app_id ->
        {:error, "Missing required line item fields: : [:bookingAmount]"}
      end)

      assert PublishInvoice.process(%{invoice: invoice}) ==
               {:error, "Missing required line item fields: : [:bookingAmount]"}
    end

    test "error with missing invoice data fields" do
      invoice =
        insert(:invoice, accounting_reference: nil, creditor_reference: nil, language_code: nil)

      assert PublishInvoice.process(%{invoice: invoice}) ==
               {:error,
                "Missing required fields: [:externalReference, :creditorReference, :accountingReference, :iban, :languageCode, :invoiceDate]"}
    end

    test "log and returns {:error, reason}", %{invoice: invoice} do
      AvkService.Boundary.AmqpPublisherMock
      |> expect(:send, fn _data, _tenant_name, _message_id, _event_type, _app_id ->
        {:error, "some error"}
      end)

      assert PublishInvoice.process(%{invoice: invoice}) == {:error, "some error"}
    end
  end
end
```

## Receive from a Queue

```elixir
defmodule AvkService.Boundary.AmqpConsumer do
  @moduledoc """
  Broadway consumer for RabbitMQ messages.
  """
  use Broadway
  require Logger

  alias AvkService.Pipeline.Worker.InvoiceResponse
  alias Jason

  @queue "avk_service"

  def start_link(_opts \\ []) do
    config = Application.fetch_env!(:avk_service, :rabbitmq)

    producer =
      {BroadwayRabbitMQ.Producer,
       queue: @queue,
       connection: config[:connection_params],
       qos: [prefetch_count: 50],
       # need list attributes to read metadata fields
       metadata: [:correlation_id],
       on_failure: :reject_and_requeue}

    Broadway.start_link(__MODULE__,
      name: __MODULE__,
      producer: [module: producer, concurrency: 1],
      processors: [default: [concurrency: 50]],
      batchers: [default: [batch_size: 10, batch_timeout: 1500, concurrency: 5]]
    )
  end

  @impl true
  # must return messages (needed?)
  def handle_batch(_batcher, messages, _batch_info, _context), do: messages

  @impl true
  def handle_message(_, message, _) do
    with {:ok, payload} when is_map(payload) <- Jason.decode(message.data),
         correlation_id when not is_nil(correlation_id) <- message.metadata.correlation_id,
         event_type when not is_nil(event_type) <- payload["eventType"],
         id when is_integer(id) <- invoice_id(correlation_id),
         data when not is_nil(data) <- payload["data"] do
      InvoiceResponse.update_invoice(event_type, id, data)
      # must return a message!
      message
    else
      _ ->
        Logger.error("Invoice response ignored: #{inspect(message)}")
        Broadway.Message.failed(message, "unprocessable")
    end
  end

  defp invoice_id("invoice-" <> str_id) do
    {id, _str} = Integer.parse(str_id)
    id
  end

  defp invoice_id(correlation_id) do
    Logger.error("Unexpected Correlation ID: #{inspect(correlation_id)}")
    nil
  end
end
```

test config
```elixir
import Config

# Configure your database
#
# The MIX_TEST_PARTITION environment variable can be used
# to provide built-in test partitioning in CI environment.
# Run `mix help test` for more information.
config :avk_service, AvkService.Repo,
  username: System.get_env("POSTGRES_USER", "rem2"),
  password: System.get_env("POSTGRES_PW", ""),
  hostname: System.get_env("POSTGRES_HOST", "localhost"),
  database: "avk_service_test#{System.get_env("MIX_TEST_PARTITION")}",
  pool: Ecto.Adapters.SQL.Sandbox,
  pool_size: System.schedulers_online() * 2

# We don't run a server during test. If one is required,
# you can enable the server option below.
config :avk_service, AvkServiceWeb.Endpoint,
  http: [ip: {127, 0, 0, 1}, port: 4002],
  secret_key_base: "p8wGTGKppAKj3N+r7iTdo9w0WfWRSeRMt7Bzv8wNzw0mvvHn1LtlolXPijRZJSdX",
  server: false

# DO NOT use rabbitmq consumer in tests (its not available)
# AvkService.Boundary.AmqpConsumer,
config :avk_service, :application_children, [
  AvkServiceWeb.Telemetry,
  AvkService.Repo,
  {DNSCluster, query: Application.get_env(:avk_service, :dns_cluster_query) || :ignore},
  {Phoenix.PubSub, name: AvkService.PubSub},
  # Start the Finch HTTP client for sending emails
  {Finch, name: AvkService.Finch},
  # Start a worker by calling: AvkService.Worker.start_link(arg)
  # {AvkService.Worker, arg},
  # Start to serve requests, typically the last entry
  AvkServiceWeb.Endpoint,
  # PromEx should be started after the Endpoint, to avoid unnecessary error messages
  AvkService.PromEx,
  # invoice pdf processing pipeline
  AvkService.Pipeline.PdfFilesQueue,
  {AvkService.Pipeline.BroadwayPdfToInvoice, []}
]

config :avk_service, :parashift_api,
  token: "token",
  endpoint: "https://api.parashift.io/v2",
  classification_scope: "garaio-pp-invoice-ch"

config :avk_service, :dti_api,
  app_key: "123",
  endpoint: "https://cloudextract-studio.dti.ch/extraction/api/",
  plan_id: "123"

config :tesla, adapter: Tesla.Mock

# Stub Req requests in tests
config :avk_service, graphql_req_options: [plug: {Req.Test, AvkService.Boundary.GraphQL}]

config :avk_service,
  cloud_extract_dti_api: [plug: {Req.Test, AvkService.Boundary.CloudextractDtiApi}],
  cloud_extract_parashift_api: [plug: {Req.Test, AvkService.Boundary.ParashiftApi}]

# In test we don't send emails.
config :avk_service, AvkService.Mailer, adapter: Swoosh.Adapters.Test

# Disable swoosh api client as it is only required for production adapters.
config :swoosh, :api_client, false

# Print only errors during test
config :logger, level: :none

# Initialize plugs at runtime for faster test compilation
config :phoenix, :plug_init_mode, :runtime

config :phoenix_live_view,
  # Enable helpful, but potentially expensive runtime checks
  enable_expensive_runtime_checks: true

config :junit_formatter,
  report_file: "report_file_test.xml",
  report_dir: "./ci_reports",
  print_report_file: true,
  prepend_project_name?: true

config :avk_service, :admin_basic_auth,
  username: "testadmin",
  password: "testpassword"

config :avk_service, :graphql_api,
  client_id: "test_client_id",
  client_secret: "test_client_secret"

# Boundary Modules where we cannot mock the API
config :avk_service, :boundaries, storage: AvkService.Boundary.StorageMock
config :avk_service, :boundaries, graphql: AvkService.Boundary.GraphQLMock
config :avk_service, :boundaries, extraction_api: AvkService.Boundary.ExtractionApiMock
config :avk_service, :boundaries, amqp_publisher: AvkService.Boundary.AmqpPublisherMock

# RabbitMQ Config
config :avk_service, :rabbitmq,
  connection_params: [
    host: "localhost",
    virtual_host: "/",
    port: 5672,
    username: "admin",
    password: "admin"
  ]
```

runtime config

```elixir
import Config

if System.get_env("PHX_SERVER") do
  config :avk_service, AvkServiceWeb.Endpoint, server: true
end

if config_env() != :test do
  config :avk_service, :application_children, [
    AvkServiceWeb.Telemetry,
    AvkService.Repo,
    {DNSCluster, query: Application.get_env(:avk_service, :dns_cluster_query) || :ignore},
    {Phoenix.PubSub, name: AvkService.PubSub},
    # Start the Finch HTTP client for sending emails
    {Finch, name: AvkService.Finch},
    # Start a worker by calling: AvkService.Worker.start_link(arg)
    # {AvkService.Worker, arg},
    # Start to serve requests, typically the last entry
    AvkServiceWeb.Endpoint,
    # PromEx should be started after the Endpoint, to avoid unnecessary error messages
    AvkService.PromEx,
    # retrieves (rabbitmq) messages returned from Invoicing.Invoice.Created events
    AvkService.Boundary.AmqpConsumer,
    # invoice pdf processing pipeline
    AvkService.Pipeline.PdfFilesQueue,
    {AvkService.Pipeline.BroadwayPdfToInvoice, []}
  ]

  config :avk_service, :boundaries, amqp_publisher: AvkService.Boundary.AmqpPublisher

  # RabbitMQ Config
  config :avk_service, :rabbitmq,
    connection_params: [
      host:
        System.get_env("RABBITMQ_HOST") ||
          raise("Missing env variable: RABBITMQ_HOST"),
      virtual_host:
        System.get_env("RABBITMQ_VHOST") ||
          raise("Missing env variable: RABBITMQ_VHOST"),
      port:
        String.to_integer(System.get_env("RABBITMQ_PORT")) ||
          raise("Missing env variable: RABBITMQ_PORT"),
      username:
        System.get_env("RABBITMQ_USER") ||
          raise("Missing env variable: RABBITMQ_USER"),
      password:
        System.get_env("RABBITMQ_PASSWORD") ||
          raise("Missing env variable: RABBITMQ_PASSWORD")
    ]
end
```

testing

```elixir
defmodule AvkService.Boundary.AmqpConsumerTest do
  use AvkService.DataCase, async: true
  import AvkService.Factory

  alias AvkService.Boundary.AmqpConsumer
  alias AvkService.Invoicing
  alias AvkService.Repo
  alias Broadway.Message

  describe "handle_message/3" do
    setup do
      now = DateTime.utc_now() |> DateTime.to_date()
      due_date = Date.add(now, 30) |> Date.to_string()
      invoice_date = Date.add(now, -10) |> Date.to_string()

      storage_filepath = Ecto.UUID.generate() <> ".pdf"

      invoice =
        insert(:invoice,
          iban: "CH9300762011623852957",
          external_reference: "123456789",
          storage_filepath: storage_filepath,
          accounting_reference: "1000",
          creditor_reference: "2345",
          order_reference: "34567",
          invoice_date: invoice_date,
          due_date: due_date,
          stage: "publish_invoice",
          state: "PUBLISHED"
        )

      insert_list(2, :invoice_line_item, invoice: invoice)
      invoice = invoice |> Repo.preload([:tenant, :line_items])

      accepted_data = %{key: "value"}
      rejected_data = %{reasons: [%{reason: "first reason"}, %{reason: "second reason"}]}

      accepted_message = %Message{
        data: Jason.encode!(%{eventType: "Invoicing.Invoice.Accepted", data: accepted_data}),
        metadata: %{correlation_id: "invoice-#{invoice.id}"},
        acknowledger: {AvkService.Boundary.AmqpConsumer, :ack_id, :ack_data}
      }

      rejected_message = %Message{
        data: Jason.encode!(%{eventType: "Invoicing.Invoice.Rejected", data: rejected_data}),
        metadata: %{correlation_id: "invoice-#{invoice.id}"},
        acknowledger: {AvkService.Boundary.AmqpConsumer, :ack_id, :ack_data}
      }

      %{invoice: invoice, accepted_message: accepted_message, rejected_message: rejected_message}
    end

    test "updates the invoice state when receiving an Invoicing.Invoice.Accepted message", %{
      invoice: invoice,
      accepted_message: accepted_message
    } do
      _processed_message = AmqpConsumer.handle_message(nil, accepted_message, nil)
      updated_invoice = Invoicing.get_invoice!(invoice.id)

      assert updated_invoice.stage == "invoice_response"
      assert updated_invoice.state == "OK"

      expected = Invoicing.get_invoice!(invoice.id)
      assert updated_invoice == expected
    end

    test "updates the invoice state when receiving an Invoicing.Invoice.Rejected", %{
      invoice: invoice,
      rejected_message: rejected_message
    } do
      _processed_message = AmqpConsumer.handle_message(nil, rejected_message, nil)
      updated_invoice = Invoicing.get_invoice!(invoice.id)

      assert updated_invoice.stage == "invoice_response"
      assert updated_invoice.state == "REJECTED"

      expected = Invoicing.get_invoice!(invoice.id)
      assert updated_invoice == expected
    end

    test "does not change the invoice state when receiving an unexpected event", %{
      invoice: invoice
    } do
      unexpected_message =
        %Message{
          data:
            Jason.encode!(%{eventType: "Invoicing.Invoice.Unexpected", data: %{key: "value"}}),
          metadata: %{correlation_id: "invoice-#{invoice.id}"},
          acknowledger: {AvkService.Boundary.AmqpConsumer, :ack_id, :ack_data}
        }

      _processed_message = AmqpConsumer.handle_message(nil, unexpected_message, nil)
      updated_invoice = Invoicing.get_invoice!(invoice.id)

      assert updated_invoice.stage == "publish_invoice"
      assert updated_invoice.state == "PUBLISHED"
    end

    test "does not change the invoice state when receiving an invalid correlation_id", %{
      invoice: invoice
    } do
      unexpected_message =
        %Message{
          data:
            Jason.encode!(%{eventType: "Invoicing.Invoice.Unexpected", data: %{key: "value"}}),
          metadata: %{correlation_id: "invoice-0"},
          acknowledger: {__MODULE__, :ack_id, :ack_data}
        }

      _processed_message = AmqpConsumer.handle_message(nil, unexpected_message, nil)
      updated_invoice = Invoicing.get_invoice!(invoice.id)

      assert updated_invoice.stage == "publish_invoice"
      assert updated_invoice.state == "PUBLISHED"
    end
  end
end
```

## Resources
