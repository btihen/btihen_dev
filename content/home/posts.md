---
# An instance of the Pages widget.
# Documentation: https://wowchemy.com/docs/page-builder/
widget: collection
active: false        # Activate this widget? true/false

# This file represents a page section.
headless: true

# Order that this section appears on the page.
weight: 20

title: Recent Posts
subtitle:

content:
  # Filter on criteria
  filters:
    folders:
      - posts
    tag: ''
    category: ''
    publication_type: ''
    author: 'btihen'
    exclude_featured: false
    exclude_future: false
    exclude_past: false
  # Choose how many pages you would like to display (0 = all pages)
  count: 3
  # Choose how many pages you would like to offset by
  offset: 0
  # Page order: descending (desc) or ascending (asc) date.
  order: desc

design:
  # Toggle between the various page layout types.
  #   1 = List
  #   2 = Compact
  #   3 = Card
  #   5 = Showcase
  view: Compact
  columns: '2'
---
