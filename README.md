trello_lead_time
=====

Trello is awesome for lightweight task/process management. It's so lightweight that you have to figure out your own way to gather analytics about your process.

*trello_lead_time* is a Ruby gem to calculate the lead time, queue time, and cycle time of cards in a Trello list.

**Queue time** - the time a Trello card spends waiting to be started.

**Cycle time** - the time a Trello card spends in progress.

**Lead time** - the time it takes a Trello card to get started and completed, e.g. queue time + cycle time.

Requirements
-----

1. Currently, only lists belonging to a Trello organization are supported
1. Uses the 'ruby-trello' gem (see https://github.com/jeremytregunna/ruby-trello/)

Usage
-----

Because *trello_lead_time* depends on *ruby-trello*, there is some configuration required.

First, load up the gem in your script.

    require 'trello_lead_time'
    
Establish your Trello credentials.

    developer_public_key = 'YOUR TRELLO PUBLIC KEY'
    member_token         = 'YOUR TRELLO MEMBER TOKEN'

Decide where to find the data. `organization_name` is the URL name that can be used to find your board by URL. You can find this organization name by navigating to your organization's page in Trello. 

The `board_url` is the URL to the actual board. Why do we need the organization name if we have the board's URL? Well, the Trello API doesn't allow us to directly find a board by its URL, so we have to go through the organization. (Finding a board's internal ID to use with the API isn't possible without first using the API to get a list of the boards!)

    organization_name    = 'fogcreek'
    board_url            = 'https://trello.com/b/nC8QJJoZ/trello-development'

Now, configure how you will calculate the metrics.

`source_lists` is an array of Trello list names (can be open or archived) that contain cards you want to analyze. This gem will compute the lead time, queue time, and cycle time for each of these lists.

`queue_time_lists` is an array of lists that represent the phase of the Trello card's life when it was waiting to start.

`cycle_time_lists` is an array of lists that represent the phase of the Trello card's life when it was actively being worked, or in progress.

    source_lists         = ['Live (4/8)', 'Live (3/17)', 'Live (3/3)', 'Live (2/11)', 'Live (1/14)']
    queue_time_lists     = ['Next Up']
    cycle_time_lists     = ['In Progress', 'Testing']

Lead time analysis can be performed on a per-label basis. Create an array of labels and for which you want to get the total lead time of cards with those labels.

    finance_type_labels        = ['Feature', 'Bug']

When analyzing the timeline of each Trello card, the gem has to find out when it was moved into a "Done" state. This usually is the event when the card is placed in the one of the `source_lists`, e.g. "Live (4/8)". 

You must provide a regular expression that will be compared to the names of lists a Trello card was in (at some point of its life) to identify when it was marked done.

    list_name_matcher_for_done = /^Live/

Now that all those variables are set, you can configure the gem!

    TrelloLeadTime.configure do |cfg|
      cfg.organization_name          = organization_name
      cfg.queue_time_lists           = queue_time_lists
      cfg.cycle_time_lists           = cycle_time_lists
      cfg.finance_type_labels        = finance_type_labels
      cfg.list_name_matcher_for_done = /^Live/
      cfg.set_trello_key_and_token(developer_public_key, member_token)
    end

Easy right? Now let's put it to use!

    puts "-" * 40
    puts "Calculating metrics for:"
    puts "#{board_url}"
    puts "-" * 40

    board = TrelloLeadTime::Board.from_url board_url
    source_lists.each do |source_list|
      totals   = board.totals(source_list)
      averages = board.averages(source_list)

      puts "Overall metrics for: #{source_list}"
      puts "\tAverage Card Age:   #{TrelloLeadTime::TimeHumanizer.humanize_seconds(averages[:age][:overall])}"
      puts "\tAverage Lead Time:  #{TrelloLeadTime::TimeHumanizer.humanize_seconds(averages[:lead_time][:overall])}"
      puts "\tAverage Queue Time: #{TrelloLeadTime::TimeHumanizer.humanize_seconds(averages[:queue_time][:overall])}"
      puts "\tAverage Cycle Time: #{TrelloLeadTime::TimeHumanizer.humanize_seconds(averages[:lead_time][:overall])}"
      puts ""
      puts "\tTotal Card Age:     #{TrelloLeadTime::TimeHumanizer.humanize_seconds(totals[:age][:overall])}"
      puts "\tTotal Lead Time:    #{TrelloLeadTime::TimeHumanizer.humanize_seconds(totals[:lead_time][:overall])}"
      puts "\tTotal Queue Time:   #{TrelloLeadTime::TimeHumanizer.humanize_seconds(totals[:queue_time][:overall])}"
      puts "\tTotal Cycle Time:   #{TrelloLeadTime::TimeHumanizer.humanize_seconds(totals[:lead_time][:overall])}"
      puts ""
      puts "\tFinance type breakdown (total lead time per label):"
      totals[:lead_time][:finance_types].each do |label, value|
          puts "\t\t#{label}: #{TrelloLeadTime::TimeHumanizer.humanize_seconds(value)}"
      end
      puts ""
    end

For each of the source lists, the cards are analyzed and aggregate (average and total) metrics are displayed. This takes some time because the Trello API must be hit for each card being analyzed. Therefore, the longer the lists in Trello, the longer it takes to process.

See [sample.rb](sample.rb) for an entire listing of the explanation above.
