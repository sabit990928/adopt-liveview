%{
title: "Simplifying everything with Ecto",
author: "Lubien",
tags: ~w(getting-started),
section: "Forms",
description: "Now things get serious",
next_page_id: "my-first-liveview-project"
}

---

Now that you understand not only how forms work behind the scenes but also how to reason through the flow of forms using events, let's simplify everything!

## Introducing Ecto

Ecto is a library in Elixir for managing database access. Over time, the community noticed that Ecto's validation pattern was quite powerful and abstractions to validate data without even considering the database were emerging. Today we will use one of them.

It's worth mentioning that in new Phoenix projects, Ecto comes by default, so understanding Ecto will not only help us today to refactor our form into a more organized code, but it will also teach you one of the fundamentals of Ecto so that you can use this library in the future in your projects. projects.

## Refactoring our previous form to Ecto

Let's get straight to the point. Create and run a file called `ecto_form.exs`:

```elixir
Mix.install([
  {:liveview_playground, "~> 0.1.8"},
  {:phoenix_ecto, "~> 4.5"},
  {:ecto, "~> 3.11"}
])

defmodule Product do
  use Ecto.Schema
  import Ecto.Changeset

  embedded_schema do
    field :name, :string, default: ""
    field :description, :string, default: ""
  end

  def changeset(product, params \\ %{}) do
    product
    |> cast(params, [:name, :description])
    |> validate_required([:name, :description])
  end
end

defmodule PageLive do
  use LiveviewPlaygroundWeb, :live_view
  import LiveviewPlaygroundWeb.CoreComponents

  def mount(_params, _session, socket) do
    form =
      Product.changeset(%Product{})
      |> to_form()

    {:ok, assign(socket, form: form)}
  end

  def handle_event("validate_product", %{"product" => product_params}, socket) do
    form =
      Product.changeset(%Product{}, product_params)
      |> Map.put(:action, :validate)
      |> to_form()

    {:noreply, assign(socket, form: form)}
  end

  def handle_event("create_product", %{"product" => product_params}, socket) do
    IO.inspect({"Form submitted!!", product_params})
    {:noreply, socket}
  end

  def render(assigns) do
    ~H"""
    <div class="bg-grey-100">
      <.form
        for={@form}
        phx-change="validate_product"
        phx-submit="create_product"
        class="flex flex-col max-w-96 mx-auto bg-gray-100 p-24"
      >
        <h1>Creating a product</h1>
        <.input field={@form[:name]} placeholder="Name" />
        <.input field={@form[:description]} placeholder="Description" />

        <.button type="submit">Send</.button>
      </.form>
    </div>
    """
  end
end

LiveviewPlayground.start(scripts: ["https://cdn.tailwindcss.com?plugins=forms"])
```

### Installing the necessary libraries

```elixir
Mix.install([
  {:liveview_playground, "~> 0.1.8"},
  {:phoenix_ecto, "~> 4.5"},
  {:ecto, "~> 3.11"}
])
```

In our `Mix.Install/2` we added not only Ecto itself but also the `phoenix_ecto` library that serves to unite both. In real projects this would already be installed, don't worry.

### Understanding an Ecto Schema

```elixir
defmodule Product do
  use Ecto.Schema
  import Ecto.Changeset

  # ...
end
```

The magic starts here. We define a module called `Product` to represent the data in our form. The first thing we do is `use Ecto.Schema` so that our module receives the DSL (Domain Specific Language) that lets us use macros like `embedded_schema` and `field` to define the format of our Product. Think of this DSL as a simple way to define a Struct in Elixir.

Additionally we import [`Ecto.Changeset`](https://hexdocs.pm/ecto/Ecto.Changeset.html). Changeset is a data structure that contains data about modifications to something. In this case, our Changeset will contain data about modifications, errors and validations of our Product struct. Think of changesets as a validation phase.

### Understanding an `embedded_schema`

```elixir
defmodule Product do
  # ...

  embedded_schema do
    field :name, :string, default: ""
    field :description, :string, default: ""
  end

  # ...
end
```

In Ecto terms, Embedded Schemas are data that live only in memory, without being saved in a database. Using the above syntax and defining the struct fields with [`field/3`](https://hexdocs.pm/ecto/Ecto.Schema.html#field/3) we can easily tell which data belongs to the Product struct. Essentially what this piece of code does is say that a Struct Product starts as `%Product{name: "", description: ""}` but with very easy to understand code.

### The `changeset/2` function

```elixir
defmodule Product do
  # ...

  def changeset(product, params \\ %{}) do
    product
    |> cast(params, [:name, :description])
    |> validate_required([:name, :description])
  end
end
```

It is practically inevitable that you will see an Ecto.Schema without a `changeset/2` function or even more than one. This function is under your full control and is used to define how we validate your data. In the previous class, validation took place within LiveView, but this left our code messy and difficult to share. In Phoenix projects, validations at the level of an Ecto.Schema are almost always carried out in this function.

In it we receive two arguments: the product and optionally new parameters (see that if nothing is passed we use the default `%{}`). With these two values in mind we use pipes to transform this value as follows:

- We have a struct of `%Product{name: "", description: ""}` (in the case of our form it starts empty).
- Using the function [`cast/4`](https://hexdocs.pm/ecto/Ecto.Changeset.html#cast/4) we transform the `%Product{}` into a `%Ecto.Changeset{}` receiving the `params` and accepting only the `params` that are `:name` or ` :description`.
- Using the function [`validate_required/3`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_required/3) we receive the `%Ecto.Changeset{}` with the data and validate that `:name` and `:description` are present.

At the end of the function we will have a validated changeset. The `Ecto.Changeset` module contains several useful validation functions and you can also create custom validations. At the moment we will only continue with `validate_required/3`.

## Using changesets in our LiveView

```elixir
defmodule PageLive do
  # ...

  def mount(_params, _session, socket) do
    form =
      Product.changeset(%Product{})
      |> to_form()

    {:ok, assign(socket, form: form)}
  end

  def handle_event("validate_product", %{"product" => product_params}, socket) do
    form =
      Product.changeset(%Product{}, product_params)
      |> Map.put(:action, :validate)
      |> to_form()

    {:noreply, assign(socket, form: form)}
  end

  # ...
end
```

Now that we have changesets, the only refactoring necessary for our LiveView happened in the callbacks. Let's analyze them step by step.

#### The new `mount/3`

```elixir
def mount(_params, _session, socket) do
  form =
    Product.changeset(%Product{})
    |> to_form()

  {:ok, assign(socket, form: form)}
end
```

In our `mount/3` we use the module's `changeset/2` function without passing the second argument as we know that there is no modified data. We immediately pass the result to `to_form/2`.

You may be wondering: don't we need an `as: :product`? Phoenix forms are prepared to automatically convert the `Ecto.Schema` name so that a `Product` schema implicitly means `as: :product` in `to_form/2`. It's worth remembering that from the beginning we mentioned that this was the Phoenix standard and you can see how the framework takes this seriously to the point of simplifying it for you.

#### The new `handle_event/3`

```elixir
def handle_event("validate_product", %{"product" => product_params}, socket) do
  form =
    Product.changeset(%Product{}, product_params)
    |> Map.put(:action, :validate)
    |> to_form()

  {:noreply, assign(socket, form: form)}
end
```

Very similar to `mount/3`, our function also uses changeset to create `Phoenix.HTML.Form`. We had two modifications:

- We pass the `product_params` to the changeset so that the new data is validated.
- We use `Map.put/3` to define in the changeset that we are in validation mode. This is necessary so that our LiveView knows that the changeset has been validated and errors can be rendered.

## In short!

- Ecto is a library for managing powerful database access.
- Phoenix projects use Ecto by default not only to work with databases but also to validate data.
- We can use `Ecto.Schema` to easily create a `Struct`. As we are not working with a database (yet) we use `embedded_schema` and `fields` to be able to create data only in memory.
- We can use `Ecto.Changeset` to easily validate user data going into your struct.
- Generally an `Ecto.Schema` has one or more `changeset/2` functions to define how to validate its data and are used as `Product.changeset(%Product{}, params)`.
- Our LiveViews become leaner when we separate the logic of validating data from our logic of rendering and receiving form events.