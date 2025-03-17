Reactive Framework for perl - Inspired by Laravel Livewire

NOT PRODUCTION READY YET

## Installing

the easiest way to install for now is with [cpm](https://metacpan.org/pod/App::cpm::Tutorial)

add
```
requires 'Reactive::Core', git => 'https://github.com/ReactivePL/Core.git';
requires 'Reactive::Mojo', git => 'https://github.com/ReactivePL/Mojo.git';
```
to your `cpanfile` and then run `cpm install -g`

or run
```
cpm install -g https://github.com/ReactivePL/Core.git
cpm install -g https://github.com/ReactivePL/Mojo.git
```

register the plugin with `Mojolicious`, providing a list of namespaces to scan for components

```PERL
    $self->plugin(
        'Reactive::Mojo::Plugin',
        {
            namespaces => [
                'Your::App::Components',
            ],
        },
    );
```

If you wish to make use of DBIx::Class objects in your properties, we need a hook for your `DBIx::Class::Schema`

eg

```PERL
use Sub::Override;
my $override = Sub::Override->new;

...
$override->replace('Reactive::Core::Types::dbic_schema', sub { $schema });
```

(this part of the API is highly likely to change as this is a pretty horrible solution but for now this is what is required)


## Your first component

```PERL
package Your::App::Components::Counter;

use Moo;
use Types::Standard qw( Int );

has count => (is => 'rw', isa => Int, default => sub { return 0 });

sub render {
    my $self = shift;

    return <<'HTML';
        <div class="counter">
            <button reactive:click.decrement="count">-</button>
            <span><%= $count %></span>
            <button reactive:click.increment="count">+</button>
        </div>
HTML
}

1;
```

* `package Your::App::Components::Counter;`

create a new module within the namescape you specified earlier

* `use Moo;`

`Reactive` components currently need to use [Moo](https://metacpan.org/pod/Moo)

* `use Types::Standard qw( Int );`

import the `Int` type from [Type::Tiny](https://metacpan.org/pod/Type::Tiny)

this part is optional but highly recommended

* `has count => (is => 'rw', isa => Int, default => sub { return 0 });`

declare a property `count` which can be read + written, it must be an integer, and default value of 0

* `sub render { ... }`

render method should return the template for rendering this component, you can use the usual Mojolicious stuffs

your component needs to have a single root element

the properties of your component will be accessible as variables

the `reactive:...` attributes are what makes the magic work, there are a few available but here we see `reactive:click.increment` and `reactive:click.decrement`, these 2 both work pretty much the same and take the name of the property you want them to act on in this case `count`

## Using Components

in your regular mojo template, you can add your new `Counter` component with

```
<%= reactive('Counter') %>
```

or if you want to provide the initial value,

```
<%= reactive('Counter', count => 42) %>
```

you will also need to add
```
<%= reactive_js %>
```
somewhere in your template, could potentially add to base layout if you are making wide use of components

## `reactive:` attributes

* `reactive:click="method"` will call `->method()` on your component, eg if you have a form you may do `reactive:click="save"` to call the `->save()` method which handles writing to the database

* `reactive:click.increment="property"` will add 1 to the current value of `$property`

* `reactive:click.decrement="property"` will subtract 1 from the current value of `$property`

* `reactive:click.unset="property"` will set the property to null/undef

* `reactive:model="property"` will bind the value of an input element to a proerty in your model

* `reactive:model.lazy="property"` will bind the value of an input element to a proerty in your model (though changes will only be synced on blur rather than while typing)


## `DBIx::Class` objects in component properties

* currently requires the `Reactive::Core::Types::dbic_schema` override hack mentioned earlier

* try to keep use of these to a minimum,

* will currently only work if your model has a PK called `id`

* use `Reactive::Core::Types::DBIx[Your::App::Schema::Result::Model]` type contraint on the property and include `coerce => 1` attribute

eg
```PERL
use Reactive::Core::Types qw( DBIx );
has post => (is => 'ro', isa => DBIx['Your::App::Schema::Result::Post'], coerce => 1);
```

* if you wish to expose data from your model to your JS, add a `short_summary` method to your model which returns a HashRef of the data you want, you should keep the data returned from this to a minimum both for performance + security. this step is only required if you want the data available to JS, your template in `render` will have access to the full object

## Hooks

there are a few hooks that you can add additional logic to

if your component has a `mounted` method, this will be called when your component is first initialised, this can be used for setting properties from a database or similar

if your component has an `updated` method, this will be called when a property is updated via `reactive:model` or `reactive:click.increment|decrement|unset`

## Other

* when rendering the template for your component, the component itself will be available as `$self`, this is mainly for calling methods as the properties are available directly, an example of a potential use of this can be seen [here](https://github.com/ReactivePL/MojoDemo/blob/d17a46b7be1051a81456681195fb84d2dc16ec68/lib/ReactivePL/Reactive/Components/DataTable.pm#L118) where it calls a method to fetch DB search results

* highly recommend having coercions set up / enabled for properties
