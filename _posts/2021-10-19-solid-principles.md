---
layout: post
title:  "On SOLID's Single Responsibility and Open/Closed Principles"
date:   2021-10-19 14:57:41 -0400
categories: blog
---

First introduced by Robert C. Martin in the early 2000s, the [SOLID principles](https://en.wikipedia.org/wiki/SOLID) have become widely used guidelines for designing object oriented programming that is easy to scale and maintain. 

SOLID is a mnemonic acronym that stands for: 

- S - Single Responsiblity Principle
- O - Open/Closed Principle
- L - Liskov Substitution Principle
- I - Interface Segregation Principle
- D - Dependency Inversion Principle

In this article, we will examine the Single Responsibility and Open/Closed principles.

**Single Responsibility Principle**

The first of the five SOLID principles, the Single Responsibility principle states that a class should only have one reason to change. This is achieved by only having a single responsibility, or a singular job to do. 

This principle serves two purposes. A class with a single responsibility is far easier to understand and implement than a class with multiple responsibilities. It is also far easier to change and will largely prevent the unexpected side-effects that are common with classes that encapsulate lots of functionality and dependencies. Take the following **Board** class from a game of Tic-Tac-Toe as an example. 

```
class Board
    attr_reader : squares, :player_1, :player_2
    
    def initialize(squares = %w[1 2 3 4 5 6 7 8 9], player_1 = "X", player_2 = "O")
        @squares = squares
        @player_1 = player_1
        @player_2 = player_2
        @current_player = player_1
    end
    
    def display_board
        puts "\n #{squares[0]} | #{squares[1]} | #{squares[2]} "
        puts "-----------"
        puts " #{squares[3]} | #{squares[4]} | #{squares[5]} "
        puts "-----------"
        puts " #{squares[6]} | #{squares[7]} | #{squares[8]}
        \n"
    end

    def current_player
        turn_count.odd? ? player_2 : player_1
    end

    def make_move
        puts "Player #{current_player}, you're up!\n"
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
    
    def turn_count
        squares.count("X") + squares.count("O")
    end
end
```

The **Board** class above initializes a new board for the game by creating the `squares` array. It holds methods that are either directly affected by or have an effect on `squares`, such as `display_board` and `mark_square`. But it also initializes players through `player_1` and `player_s2`. At first glance, this might not seem like a bad idea. Our `make_move` method needs the player's information to prompt the next player to make their move. But what if we wanted to create a player for a different game - let’s call it Zic-Zac-Zoe - that had a different board set up? We wouldn’t be able to use our existing player code because it’s embedded within the Tic-Tac-Toe **Board** class so we’d have to repeat all applicable player code within Zic-Zac-Zoe. If, instead, we had an independent **Player** class, we could simply create instances of that class wherever they were needed.

```
require 'player'

class Board
    attr_reader :squares, :player_1, :player_2
    
    def initialize(squares = %w[1 2 3 4 5 6 7 8 9])
        @squares = squares
        @player_1 = Player.new(1, “X”)
        @player_2 = Player.new(2, “O”)
    end
    
    def display_board
        puts "\n #{squares[0]} | #{squares[1]} | #{squares[2]} "
        puts "-----------"
        puts " #{squares[3]} | #{squares[4]} | #{squares[5]} "
        puts "-----------"
        puts " #{squares[6]} | #{squares[7]} | #{squares[8]}
        \n"
    end

    def current_player
        turn_count.odd? ? player_2 : player_1
    end

    def make_move
        puts "Player #{current_player}, you're up!\n"
        user_input = gets.chomp
        valid_move?(user_input, current_player)
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

```
class Board
    ...
    
    def turn
        player_move = current_player.make_move
        valid_move?(player_move, current_player.player_symbol)
    end

    def valid_move?(move, symbol)
        if within_range?(move) && !square_taken?(move)
            mark_square(move, player.symbol)
        else
            false
        end
    end
    
    ...
end
```

The idea of single responsibility can be usefully employed in other parts of your code. Methods, like classes, should also have a single responsibility.  Let’s take a look at some of the methods from our  Tic-Tac-Toe **Board** class. 

```
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

All three of the above methods are taking on more than their given responsibility. Each one is converting a player's move, or input, into an integer *and* fulfilling their particular requirement. This is unnecessarily repeating code when we could simply extract that functionality out into its own method and reuse when necessary. 

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

**Open/Closed Principle**

The Open/Closed principle states that software entities such as classes, modules, functions, etc. should be open for extension, but closed for modification. This means that any new functionality should be implemented by adding new classes, attributes, and methods instead of changing existing ones.

The Open/Closed principle is highly related to the Single Responsibility principle as it maintains that classes should not take on additional functionality (read: responsibilities) that could be handled by a different class. At its root, the Open/Closed principle is saying that instead of changing code, we need to write new code, and that new code is going to work with our old code to provide the functionality that we need. 

Let’s take a look at our game of Tic-Tac-Toe. For now, we’ve only implemented functionality for a human versus human game, but what if we wanted to add a computer player? If we added the computer player to our **Player** class, we’d have to make various modifications to our **Player** class.

```
class Player

    attr_accessor :player_type, :symbol
    
    def initialize(player_type, symbol)
        @player_type = player_type
        @symbol = symbol
    end
    
    def make_move
        puts "Player #{symbol}, you're up!\n"
        move = nil
        if player_type == computer
            computer_input = rand (1...9)
        else
            move = gets.chomp
        end
        move
   end
end
```

The above changes to the **Player** class would qualify as a violation of the Open/Closed principle. We have changed the way a player is initialized and now we need to make sure that Player instances are created correctly elsewhere in our code. These changes have the potential to break our code. A better way to add a computer player would be to create a separate **Computer** class and then initialize our second player as an instance of that class.

```
class Computer
    attr_accessor :player_num, :player_symbol

    def initialize(player_num, player_symbol)
        @player_num = player_num
        @player_symbol = player_symbol
    end

    def computer_input
        rand(1..9)
    end

    def make_move
        move = computer_input
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

    def make_move
        puts "Player #{player_symbol}, you're up!\n"
        user_input = gets.chomp
    end
end
```

```
class Board
    attr_reader :squares, :player_1, :player_2
    
    def initialize(squares = %w[1 2 3 4 5 6 7 8 9])
        @squares = squares
        @player_1 = Player.new(1, “X”)
        @player_2 = Computer.new(2, “O”)
    end
end
```

In future iterations of this code, we would likely have to validate the computer's move within the `make_move` method to make sure we were returning a valid move to the board without having to prompt the computer player continously until a valid move was generated from the `computer_input` method. To do this, we could further apply the Open/Closed principle and create a **Validator** class. This would cause some modifications to our code, but it would largely prevent future validations from causing disruptive changes outside of the **Validator** class. 

> I have not implemented computer versus human in my TTT project yet, but once I do, I can add the code to this blog post to completely illustrate these changes. 