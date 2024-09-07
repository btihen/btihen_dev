---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Ruby 3.x - Lazy Evaluation, prioritized algorithms"
subtitle: "Prioritized Algorithms using lazy eval filter diverse data"
summary: ""
authors: ['btihen']
tags: ['Ruby', "Lazy", "Enumerable"]
categories: ["Code", "Lazy", "Ruby Language", "Enumerable", "Filters"]
date: 2024-09-07T01:20:00+02:00
lastmod: 2024-09-07T01T01:20:00+02:00
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

We had a problem to the 'best' fitting object within an array.  In our case it was to find the object that would return the best metadata.  But finding the 'best' person to talk with is a similar problem.

Let's say we have a list of employees and we want to find the best person to talk with about purchasing our product.

```ruby
class Person
  attr_reader :name, :rank, :role, :years_of_service

  def initialize(name, rank, role, years_of_service)
		@name = name
		@rank = rank
		@role = role
		@years_of_service = years_of_service
	end
end

people = [
	Person.new('Nyima', 3, 'Contributor', 12),
	Person.new('Marpo', 10, 'Executive', 12),
	Person.new('Seng', 8, 'Finances', 8),
	Person.new('Ziji', 4, 'Purchaser', 8),
	Person.new('Pema', 5, 'Purchaser', 12),
	Person.new('Shiné', 5, 'Purchaser', 19),
]
```

Until recently we had something like:

**Original Eval**

```ruby
module ContactSearch
	def self.purchase_contact_original_ifs(people)
	  people = Array(people).compact
		return nil if people.empty?
		return people.first if people.one?

		priority_group = people.select { _1.role == "Purchaser" }
		priority_group = people.select { _1.role == "Manager" } if priority_group.empty?
		priority_group = people.select { _1.role == "Contributor" } if priority_group.empty?
		priority_group = people.select { _1.role == "Finances" } if priority_group.empty?
		priority_group = people.select { _1.role == "Executive" } if priority_group.empty?
		priority_group = people if priority_group.empty?

		# we want the highest ranking person within our priority group
		priority_group.max_by { [_1.rank, _1.years_of_service] }
	end
end

# we expect 'Shiné'
contact = ContactSearch.purchase_contact_original_ifs(people)
```

I asked in the code review if anyone had ideas to refactor / improve this code.

**Intermediate Lazy Eval**

My colleague 'Oli' suggested experimentation with `lazy` eval - given we are working with Enumerable filters.  So on the first try we used:

```ruby
module ContactSearch
	def self.purchase_contact_first_lambda(people)
	  people = Array(people).compact
		return nil if people.empty?
		return people.first if people.one?

		prioritized_search = [
			-> { people.select { _1.role == "Purchaser" } },
			-> { people.select { _1.role == "Manager" } },
			-> { people.select { _1.role == "Contributor" } },
			-> { people.select { _1.role == "Finances" } },
			-> { people.select { _1.role == "Executive" } },
			-> { people }
		]

		priority_group = prioritized_search.lazy.map(&:call).detect { !_1.empty? }

		# we want the highest ranking person within our priority group
		priority_group.max_by { [_1.rank, _1.years_of_service] }
	end
end

# we expect 'Shiné'
contact = ContactSearch.purchase_contact_first_lambda(people)
```

Unfortunately, with benchmarking, I noticed lazy eval didn't work as expected (it seemed to be work with eager eval instead of lazy eval).  _See the Benchmarking section._

**Final Lazy Eval**

On a whim, I tried another refactor using lambda's (with an input)

```ruby
module ContactSearch
	PRIORITIZED_SELECTORS = [
		->(people) { people.select { _1.role == "Purchaser" } },
		->(people) { people.select { _1.role == "Manager" } },
		->(people) { people.select { _1.role == "Contributor" } },
		->(people) { people.select { _1.role == "Finances" } },
		->(people) { people.select { _1.role == "Executive" } },
		->(people) { people }
		].freeze

	def self.purchase_contact_second_lambda(people)
	  people = Array(people).compact
		return nil if people.empty?
		return people.first if people.one?

		priority_group =
		  PRIORITIZED_SELECTORS.lazy.map { _1.call(people) }.detect { !_1.empty? }

		# we want the highest ranking person within our priority group
		priority_group.max_by { [_1.rank, _1.years_of_service] }
	end
end

# we expect 'Shiné'
contact = ContactSearch.purchase_contact_second_lambda(people)
```

Amazingly, this seems to be about as fast as the original, easy to read, and you can freely reorganize the lambdas without any change but in the order, thus they are completely decoupled!

## Benchmarking

Here is a very simple benchmark test.

```ruby
require 'benchmark'

Benchmark.bmbm do |x|
  x.report('Original ifs') { ContactSearch.purchase_contact_original_ifs(people)}
  x.report('First lambda') { ContactSearch.purchase_contact_first_lambda(people) }
  x.report('Second lambda') { ContactSearch.purchase_contact_second_lambda(people) }
end
#                   user     system      total        real
# Original ifs   0.000022   0.000002   0.000024 (  0.000019)
# First lambda   0.000033   0.000001   0.000034 (  0.000034)
# Second lambda  0.000015   0.000001   0.000016 (  0.000016)

Benchmark.bm do |x|
  x.report('Original ifs') { ContactSearch.purchase_contact_original_if(people)}
  x.report('First lambda') { ContactSearch.purchase_contact_oli_lambda(people) }
  x.report('Second lambda') { ContactSearch.purchase_contact_bill_lambda(people) }
end
# Rehearsal
# ------------------------------------------------
# Original ifs    0.000012   0.000004   0.000016 (  0.000012)
# First lambda    0.000016   0.000000   0.000016 (  0.000017)
# Second lambda   0.000007   0.000000   0.000007 (  0.000007)
# ------------------------------------- total: 0.000039sec
#                    user     system      total        real
# Original ifs    0.000012   0.000000   0.000012 (  0.000010)
# First lambda    0.000026   0.000001   0.000027 (  0.000024)
# Second lambda   0.000019   0.000000   0.000019 (  0.000017)
```

## Resources

* [Why and how to use lazy evaluation in Ruby?](https://github.com/ccniuj/tilap/issues/9)
* [Enumerator::Lazy (Ruby 2.7.0)](https://ruby-doc.org/core-2.7.0/Enumerator/Lazy.html)
* [Lazy Evaluation: may make code more expressive without losing on efficiency: Part I](http://blog.agiledeveloper.com/2016/06/lazy-evaluation-may-make-code-more.html)
* [Lazy Evaluation: may make code more expressive without losing on efficiency: Part II](http://blog.agiledeveloper.com/2016/06/lazy-evaluation-may-make-code-more_15.html)
* [GoRuCo 2013 - Functional Programming and Ruby by Pat Shaughnessy](https://www.youtube.com/watch?v=5ZjwEPupybw)
* [Ruby 2.0 Works Hard So You Can Be Lazy](https://patshaughnessy.net/2013/4/3/ruby-2-0-works-hard-so-you-can-be-lazy)
* [Implementing Lazy Enumerables in Ruby](https://www.sitepoint.com/implementing-lazy-enumerables-in-ruby/)
