---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/
title: "Mailroom - handling international emails"
subtitle: "As of mailroom 0.6.0 we can safely process multilingual emails (from multiple operating systems)"
# Summary for listings and search engines
summary: "atimberlake (https://hex.pm/users/atimberlake) has enabled mailroom (https://github.com/andrewtimberlake/mailroom) the ability to handle multilingual and non-UTF8 encodings.  This is particularly important when working with Windows Outlook Clients and Outlook Office 365."
authors: ["btihen"]
tags: ["Elixir", "Phoenix", "Email", "Multi-Lingual", "International", "Encoding", "Outlook", "Office365"]
categories: ["Code", "Elixir Language", "Email", "IMAP"]
date: 2025-10-17T01:01:53+02:00
lastmod: 2025-10-17T01:01:53+02:00
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

## Problem

Living in Switzerland we handle emails in 4 languages and several encoding methods (Windows, Mac ans Linux) Mail clients.

## Solution

To handle reading imap emails coming in from Multiple languages (especially from Outlook & Office 365), we need to handle multiple encoding systems (for non-ASCII characters like, ß, ç, ü, œ, etc. common in non-English languages).  This can be handled by passing parsing options in the config:
`parser_opts: [charset_handler: &handle_charset/2]`

Of course you need to character set translation code. An example is shown below:

```elixir
defmodule MyProject.Boundary.ImapClient do
  use Mailroom.Inbox
  require Logger

  alias Mailroom.Inbox.MessageContext

  def config(_opts) do
    # add this line to handle character parsing
    [parser_opts: [charset_handler: &handle_charset/2]]
    |> Keyword.merge(Application.get_env(:my_project, :mailroom, []))
  end

  match do
    fetch_mail
    process(__MODULE__, :process_email)
  end

  @doc
  """
  Common Imap Return Patterns

  - **`:seen`** - Email successfully read and processed
  - **`:flagged`** - Email processed but with a problem (i.e. not authorized)
  - **`:deleted`** - Email that should be removed
  - **`nil`** or **`:unseen`** - Email unchanged (will read again)
  - **`:answered`** - Email has been "responded" to
  """
  def process_email(%MessageContext{message: message}) do
    with {:ok, email_address} <- get_from_address(message),
         :ok <- verify_authorized(email_address),
         attachments <- Mail.get_attachments(message) do
      do_your_stuff(email_address, attachments)
    else
      {:error, reason} -> Logger.error(reason)
    end
  end

  # Outlook Client and Office365 using "iso-8859-1" for non-ASCII character by default
  defp handle_charset("iso-8859-1", string),
    do: :unicode.characters_to_binary(string, :latin1, :utf8)

  defp handle_charset(charset_name, string) when charset_name in ["utf-8", "UTF-8"] do
    case :unicode.characters_to_binary(string, :utf8) do
      binary when is_binary(binary) ->
        binary

      {:error, _binary, _rest_data} ->
        Logger.warning("ImapClient: Invalid UTF-8 string detected, replacing invalid characters")
        replace_invalid(string, <<0xFFFD::utf8>>)

      {:incomplete, _binary, _incomplete_tail} ->
        Logger.warning("ImapClient: Incomplete UTF-8 string detected, replacing invalid tail")
        replace_invalid(string, <<0xFFFD::utf8>>)
    end
  end

  # Handle unexpected charsets and ensure no invalid characters
  defp handle_charset(charset_name, string) do
    Logger.warning(
      "ImapClient: Unexpected charset: #{charset_name}, attempting to fix invalid characters"
    )

    # ensure that unexpected charsets have no invalid characters
    # if so replace with a chosen valid character `�`(<<0xFFFD::utf8>>), "?", "_", etc.
    replace_invalid(string, <<0xFFFD::utf8>>)
  end

  # safely replace a string with possible invalid characters with a placeholder
  # code from: https://github.com/andrewtimberlake (author of mailroom)
  # https://github.com/andrewtimberlake/mailroom/pull/24
  # this approach deals with large binaries efficiently
  defp replace_invalid(binary, replacement) do
    replace_invalid(binary, binary, 0, 0, [], replacement)
  end

  defp replace_invalid(<<>>, original, offset, len, acc, _replacement) do
    acc = [acc, binary_part(original, offset, len)]
    IO.iodata_to_binary(acc)
  end

  defp replace_invalid(<<char::utf8, rest::binary>>, original, offset, len, acc, replacement) do
    char_len = byte_size(<<char::utf8>>)
    replace_invalid(rest, original, offset, len + char_len, acc, replacement)
  end

  defp replace_invalid(<<_, rest::binary>>, original, offset, len, acc, replacement) do
    acc = [acc, binary_part(original, offset, len), replacement]
    replace_invalid(rest, original, offset + len + 1, 0, acc, replacement)
  end

  defp get_from_address(message) do
    case Mail.Message.get_header(message, "from") do
      {_name, email_address} when is_binary(email_address) ->
        {:ok, clean_email_address(email_address)}

      email_address when is_binary(email_address) ->
        {:ok, clean_email_address(email_address)}

      [email_address] when is_binary(email_address) ->
        {:ok, clean_email_address(email_address)}

      # rare but possible (according to RFC 5322) - to have multiple addresses in from header
      [email_address | rest] when is_binary(email_address) and length(rest) > 0 ->
        Logger.warning(
          "Mail.Imap: Multiple addresses in 'from' header, using first: #{inspect([email_address | rest])}"
        )

        {:ok, clean_email_address(email_address)}

      unexpected ->
        message =
          "Mail.Imap: cannot decode from_email #{inspect(unexpected)}, from email message: #{inspect(message)}"

        Logger.error(message)

        {:error, message}
    end
  rescue
    ArgumentError ->
      message =
        "Mail.Imap: cannot decode from_email from email message: invalid format - #{inspect(message)}"

      Logger.error(message)

      {:error, message}
  end

  # Remove angle brackets from email addresses
  # (e.g., "<email@example.com>" -> "email@example.com")
  defp clean_email_address(email) when is_binary(email) do
    email
    |> String.trim()
    |> String.trim_leading("<")
    |> String.trim_trailing(">")
  end
end
```

and a sample auth-config:
```elixir
# config/runtime.exs
  config :my_project, :mailroom,
    ssl: true,
    ssl_opts: [verify: :verify_none],
    server: System.fetch_env!("IMAP_SERVER"),
    username: System.fetch_env!("IMAP_USERNAME"),
    password: System.fetch_env!("IMAP_PASSWORD"),
    folder: :inbox,
    debug: false
```

## Resources

* [mailroom repo](https://github.com/andrewtimberlake/mailroom)
* [mailroom hexdocs](https://hexdocs.pm/mailroom/Mailroom.html)
* [Multilingual Email Handling](https://github.com/andrewtimberlake/mailroom/blob/main/docs/multi_lingual_emails.md)
