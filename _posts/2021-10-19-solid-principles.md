---
layout: post
title:  "On SOLID's Single Responsibility and Open-Closed Principles"
date:   2021-10-19 14:57:41 -0400
categories: blog
---

First introduced by Robert C. Martin in the early 2000s, the [SOLID principles](https://en.wikipedia.org/wiki/SOLID) have become widely used guidelines for designing object oriented programming that is easy to scale and maintain. 

SOLID is a mnemonic acronym that stands for: 

- S - Single-Responsiblity Principle
- O - Open-Closed Principle
- L - Liskov Substitution Principle
- I - Interface Segregation Principle
- D - Dependency Inversion Principle

In this article, we will examine the Single Responsibility and Open-Closed principles.

**Single Responsibility Principle**

The first of the five SOLID principles, the Single Responsibility principle states that a class should only have one reason to change. This is achieved by only having a single responsibility, or a singular job to do. 

This principle serves two purposes. A class with a single responsibility is far easier to understand and implement than a class with multiple responsibilities. It is also far easier to change and will largely prevent the unexpected side-effects that are common with classes that encapsulate lots of functionality and dependencies. Take the following **Board** class from a game of Tic-Tac-Toe as an example. 

```
class Board
    attr_reader : squares, :player_num, :player_symbol
    
    def initialize(squares = %w[1 2 3 4 5 6 7 8 9], player_num, player_symbol)
        @squares = squares
	    @player_num = player_num
        @player_symbol = player_symbol
    end
    
    def display_board
        puts "\n #{squares[0]} | #{squares[1]} | #{squares[2]} "
        puts "-----------"
        puts " #{squares[3]} | #{squares[4]} | #{squares[5]} "
        puts "-----------"
        puts " #{squares[6]} | #{squares[7]} | #{squares[8]}
        \n"
    end

    def make_move
        puts "Player #{player_symbol}, you're up!\n"
        user_input = gets.chomp
    end

    def valid_move?(move, symbol)
        if within_range?(move) && !square_taken?(move)
            mark_square(move, symbol)
        else
            false
        end
    end

    def within_range?(move)
        move = move.to_i
        move > 0 && move <= 9
    end

    def square_taken?(move)
        move = move.to_i
        squares[move - 1] == "X" || squares[move - 1] == "O"
    end

    def mark_square(move, player)
        move = move.to_i
        squares[move - 1] = player
    end
end
```

The **Board** class above initializes a new board for the game by creating the `squares` array. It holds methods that are either directly affected by or have an effect on `squares`, such as `display_board` and `mark_square`. But it also initializes a player through `player_num` and `player_symbol`. On first glance, this might not seem like a bad idea. Our `make_move` method needs player information to prompt the next player to make their move. But what if we wanted to create a player for a different game - let’s call it Zic-Zac-Zoe - that had a different board set up? We wouldn’t be able to use our existing player code because it’s embedded within the Tic-Tac-Toe **Board** class so we’d have to repeat all applicable player code within Zic-Zac-Zoe. If, instead, we had an independent **Player** class, we could simply create instances of that class wherever they were needed.

```
require 'player'

class Board
    attr_reader :squares, :player
    
    def initialize(squares = %w[1 2 3 4 5 6 7 8 9])
        @squares = squares
        @player = Player.new(1, “X”)
    end
    
    def display_board
        puts "\n #{squares[0]} | #{squares[1]} | #{squares[2]} "
        puts "-----------"
        puts " #{squares[3]} | #{squares[4]} | #{squares[5]} "
        puts "-----------"
        puts " #{squares[6]} | #{squares[7]} | #{squares[8]}
        \n"
    end

    def make_move
        puts "Player #{player_symbol}, you're up!\n"
        user_input = gets.chomp
    end

    def valid_move?(move, symbol)
        if within_range?(move) && !square_taken?(move)
            mark_square(move, player.symbol)
        else
            false
        end
    end

    def within_range?(move)
        move = move.to_i
        move > 0 && move <= 9
    end

    def square_taken?(move)
        move = move.to_i
        squares[move - 1] == "X" || squares[move - 1] == "O"
    end

    def mark_square(move, player)
        move = move.to_i
        squares[move - 1] = player
    end
end
```

```
class Player
    attr_accessor :player_num, :player_symbol
    
    def initialize(player_num, player_symbol)
        @player_num = player_num
        @player_symbol = player_symbol
    end
end
```

Separating these two classes makes a lot of sense given they deal with two separate entities (board and players). Consider why we thought we needed the **Board** class to handle players in the first place: the `make_move` method uses player information. But does the `make_move` method actually belong in the **Board** class? Given that a player makes a move, not the board, probably not. It should be moved to the **Player** class.

```
class Player
    attr_accessor :player_num, :player_symbol
    
    def initialize(player_num, player_symbol)
        @player_num = player_num
        @player_symbol = player_symbol
    end

    def make_move
        puts "Player #{player_symbol}, you're up!\n"
        user_input = gets.chomp
    end
end
```
The idea of single responsibility can be usefully employed in other parts of your code. Methods, like classes, should also have a single responsibility.  Let’s take a look at some of the methods from our  Tic-Tac-Toe **Board** class. 

```
    def valid_move?(move, symbol)
        if within_range?(move) && !square_taken?(move)
            mark_square(move, player.symbol)
        else
            false
        end
    end

    def within_range?(move)
        move = move.to_i
        move > 0 && move <= 9
    end

    def square_taken?(move)
        move = move.to_i
        squares[move - 1] == "X" || squares[move - 1] == "O"
    end

    def mark_square(move, player)
        move = move.to_i
        squares[move - 1] = player
    end
```

All three of the above methods are taking on more than their given responsibility. Each one is converting a player's move, or input, into an integer AND fulfilling their particular job. This is unnecessarily repeating code when we could simply extract that functionality out into its own method and reuse when necessary. 

```
    def covert_to_integer(move_string)
        move_number = move_string.to_i
    end

    def valid_move?(move, symbol)
        move = convert_to_integer(move)
    
        if within_range?(move) && !square_taken?(move)
            mark_square(move, player.symbol)
        else
            false
        end
    end

    def within_range?(move)
        move > 0 && move <= 9
    end

    def square_taken?(move)
        squares[move - 1] == "X" || squares[move - 1] == "O
    end

    def mark_square(move, player)
        squares[move - 1] = player
    end
```
***

**Open-Closed Principle**
