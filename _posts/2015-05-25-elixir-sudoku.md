---
  title: Elixir Sudoku
  date: 2015-05-25
  categories: elixir
---

Last week I was reading my friends blog [Robot Dan](http://www.robotdan.com/)
and came across his [sudoku post](http://www.robotdan.com/posts/22).

I've never written a sudoku solver before, naturally I thought of writing it in
Elixir. I spent the evening playing around trying to come up with something that
was at least somewhat idomatic.

My first solution was a very similar implementation to what Dan had on his site.
It works ok. I found a harder board that made it take a bit longer.
Interestingly I found that the memory usage was flat for easy or hard. I'm
guessing that I got the tail recursion correct.

{% highlight elixir %}

defmodule Sudoku do
  def easy, do: Sudoku.solve(Sudoku.easy_board)

  # Just using 0 rather than nil for display purposes
  def easy_board do
    [
      [8,3,0, 1,0,0, 6,0,5],
      [0,0,0, 0,0,0, 0,8,0],
      [0,0,0, 7,0,0, 9,0,0],

      [0,5,0, 0,1,7, 0,0,0],
      [0,0,3, 0,0,0, 2,0,0],
      [0,0,0, 3,4,0, 0,1,0],

      [0,0,4, 0,0,8, 0,0,0],
      [0,9,0, 0,0,0, 0,0,0],
      [3,0,2, 0,0,6, 0,4,7]
    ]
  end

  def solve(board) do
    case solve(board, 0, 0) do
      { :ok, solved_board } -> print_board(solved_board)
      { :error } -> IO.puts("Board is not valid")
    end
  end

  def solve(board, x, _) when x==9, do: { :ok, board }
  def solve(board, x, y) when y==9, do: solve(board, x + 1, 0)

  def solve(board, x, y) do
    if Enum.at(Enum.at(board, x), y) == 0 do
      solve(board, x, y, possible_values(board, x, y))
    else
      solve(board, x, y + 1)
    end
  end

  def solve(_, _, _, []), do: { :error }

  def solve(board, x, y, [head | tail]) do
    new_row = Enum.at(board, x) |> List.replace_at(y, head)
    new_board = List.replace_at(board, x, new_row)

    case solve(new_board, x, y + 1) do
      { :ok, solved_board } -> { :ok, solved_board }
      { :error } -> solve(board, x, y, tail)
    end
  end

  def possible_values(board, x, y) do
    vals = [values_in_row(board, x), values_in_column(board, y), values_in_cell(board, x, y)]
            |> List.flatten
            |> Enum.uniq

    (1..9) |> Enum.reject(&(Enum.member?(vals, &1)))
  end

  def values_in_row(board, x), do: Enum.at(board, x)
  def values_in_column(board, y), do: Enum.map(board, &(Enum.at(&1, y)))

  def values_in_cell(board, x, y) do
    start_x = div(x, 3) * 3
    start_y = div(y, 3) * 3
    Enum.slice(board, start_x, 3) |> Enum.map(&(Enum.slice(&1, start_y, 3))) |> List.flatten
  end

  def print_board([]), do: nil
  def print_board([row | rest]) do
    IO.inspect(row)
    print_board(rest)
  end
end
{% endhighlight %}

I wanted to do something a little more Elixiry though. When the recursive
function selects all possible values for a cell, and then recurses, that's the point that I thought
I might be able to shoot out a bunch of processes and do all the calculations in
parallel. A branching recursive parallel implementation. Sounded fun!

I implemented it using Elixir Tasks and process dictionaries.

Unfortunately I went wrong somewhere. I ran out of processes, or simply timed
out. I even increased the number of processes to 5M without helping. It maxed
out in RAM, CPU and times out.

I think I'll stick with the recursive solution, but I've included the other
below.

{% highlight elixir %}
# The only board that works with this one on my machine is the easy.
# The others simply timeout.
defmodule PSudoku do
  def easy, do: PSudoku.solve(PSudoku.easy_board)

  # Just using 0 rather than nil for display purposes
  def easy_board do
    [
      [8,3,0, 1,0,0, 6,0,5],
      [0,0,0, 0,0,0, 0,8,0],
      [0,0,0, 7,0,0, 9,0,0],

      [0,5,0, 0,1,7, 0,0,0],
      [0,0,3, 0,0,0, 2,0,0],
      [0,0,0, 3,4,0, 0,1,0],

      [0,0,4, 0,0,8, 0,0,0],
      [0,9,0, 0,0,0, 0,0,0],
      [3,0,2, 0,0,6, 0,4,7]
    ]
  end

  def solve(board) do
    case solve(board, 0, 0) do
      { :ok, solved_board } -> print_board(solved_board)
      { :error } -> IO.puts("ERRRO")
    end
  end

  def solve(board, x, _) when x==9, do: { :ok, board }
  def solve(board, x, y) when y==9, do: solve(board, x + 1, 0)

  def solve(board, x, y) do
    if Enum.at(Enum.at(board, x), y) == 0 do
      vals = possible_values(board, x, y)
      if length(vals) > 0 do
        result = Enum.map(vals, fn(v) -> Task.async(fn -> solve(board, x, y, v) end) end)
        |> Enum.map(&(Task.await(&1, 30_000)))
        |> Enum.find({ :error }, &match?({ :ok, _ }, &1))
      else
        { :error }
      end
    else
      solve(board, x, y + 1)
    end
  end

  def solve(board, x, y, v) when is_integer(v) do
    new_row = Enum.at(board, x) |> List.replace_at(y, v)
    new_board = List.replace_at(board, x, new_row)
    solve(new_board, x, y + 1)
  end

  def possible_values(board, x, y) do
    vals = [values_in_row(board, x), values_in_column(board, y), values_in_cell(board, x, y)]
            |> List.flatten
            |> Enum.uniq

    (1..9) |> Enum.reject(&(Enum.member?(vals, &1)))
  end

  def values_in_row(board, x), do: Enum.at(board, x)
  def values_in_column(board, y), do: Enum.map(board, &(Enum.at(&1, y)))

  def values_in_cell(board, x, y) do
    start_x = div(x, 3) * 3
    start_y = div(y, 3) * 3
    Enum.slice(board, start_x, 3) |> Enum.map(&(Enum.slice(&1, start_y, 3))) |> List.flatten
  end

  def print_board([]), do: nil
  def print_board([row | rest]) do
    IO.inspect(row)
    print_board(rest)
  end
end
{% endhighlight %}

