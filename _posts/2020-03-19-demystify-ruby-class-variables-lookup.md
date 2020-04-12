# Demystify Ruby Class Variables Lookup

ZHENG Piao-Dan @ 2020-03-19

## Preface

A post caught my attention recently. It's at [https://ruby-china.org/topics/39618](https://ruby-china.org/topics/39618). It's about class variables collecting and lookup in Ruby. And I decided to find out what's happening under the hood.

## Issue Description

See how `class_variables` and `class_variable_get` behave differently in sample:

```ruby
module Foo; @@foo = 'foo'; end
class Bar; extend Foo; end

RUBY_VERSION
# => "2.6.5"
Bar.class_variables
# => []
Bar.singleton_class.class_variables
# => [:@@foo]
Bar.singleton_class.class_variable_get(:@@foo)
# NameError (uninitialized class variable @@foo in #<Class:Bar>)
Bar.singleton_class.singleton_class.class_variable_get(:@@foo)
# => "foo"
```

## Details Investigation

NOTE: all code analyses are based on Ruby 2.6.5 if ruby version not specified.

### Related issue report and code fix

* The issue was reported back in 2013 at [https://bugs.ruby-lang.org/issues/8297](https://bugs.ruby-lang.org/issues/8297).

* Jeremy Evans made a fix in 2019, merged in Ruby 2.7.0, at [https://github.com/ruby/ruby/pull/2478](https://github.com/ruby/ruby/pull/2478).

* In our case here, Evans' fix makes `Bar.singleton_class.class_variables` returns `[]` instead of `[:@@name]`.

### How extend method works

In short, ruby's `extend` puts module as direct ancestor to the singleton class of current class, while `include` puts module as direct ancestor to current class.

In long, I don't have the long story right now...I'll tell you all about it when I dig it again.

### How class_variables method works

*Let `ST = Bar.singleton_class` for convenience.*

#### Ruby 2.6.5 behavior

For Ruby 2.6, when `Bar.class_variables` or `ST.class_variables`is called, Ruby will collect class variables defined on:

* receiver itself , `Bar` or `ST`

* **receivers' ancestors** , `Bar.ancestors` or `ST.ancestors`

Normal class and singleton class behave the same. `ST.ancestors` includes `Foo`, so `ST.class_variables` collects `[:@@foo]` from `Foo`.

#### Ruby 2.7.0 behavior

After Evans' fix, **things changed for singleton class**. Ruby will collect class variables for `ST.class_variables` from:

* singleton class itself, `ST`

* **instance class of singleton class**, `Bar`, see `cvar_front_klass` below

* **ancestors of instance class of singleton class**, `Bar.ancestors`

Note that `Bar.ancestors` doesn't include `Foo`, so `ST.class_variables` couldn't find `@@foo` and returns `[]`.

#### Take a trip into Evans' code fix

Let's take a deeper look at Evans' fix (section with `+`):

```c
/* file: variable.c */
  static void*
 mod_cvar_of(VALUE mod, void *data)
 {
     VALUE tmp = mod;
+    if (FL_TEST(mod, FL_SINGLETON)) {
+        if (rb_namespace_p(rb_ivar_get(mod, id__attached__))) {
+            data = mod_cvar_at(tmp, data);
+            tmp = cvar_front_klass(tmp);
+        }
+    }
     for (;;) {
        data = mod_cvar_at(tmp, data);
        tmp = RCLASS_SUPER(tmp);
        if (!tmp) break;
     }
     return data;
 }
```

The `if` section was added to use `cvar_front_klass` of singleton class for later class variables lookup. Here's the `cvar_front_klass`:

```c
/* file: variable.c */
static VALUE
cvar_front_klass(VALUE klass)
{
    if (FL_TEST(klass, FL_SINGLETON)) {
        VALUE obj = rb_ivar_get(klass, id__attached__);
        if (rb_namespace_p(obj)) {
            return obj;
        }
    }
    return RCLASS_SUPER(klass);
}
```

From the code we can see:

* *If `klass` is singleton class, `cvar_front_klass` would return the attached object of the singleton class.*

* **For normal class, it just returns ancestor class.**

Remember these two behaviors. Below is how Ruby attach object to singleton clss:

```c
/* file: class.c */
/*!
 * Attach a object to a singleton class.
 * @pre \a klass is the singleton class of \a obj.
 */
void
rb_singleton_class_attached(VALUE klass, VALUE obj)
{
    if (FL_TEST(klass, FL_SINGLETON)) {
        if (!RCLASS_IV_TBL(klass)) {
            RCLASS_IV_TBL(klass) = st_init_numtable();
        }
        rb_class_ivar_set(klass, id_attached, obj);
    }
}
```

From the code we can know that **the attached object of a singleton class is actually a class itself**. According to this, `cvar_front_klass` for `Bar.singleton_class` is `Bar`. That's why `ST.class_variables` in Ruby 2.7.0 collects class variables from `Bar.ancestors` not `ST.ancestors`

### How class_variable_get method works

*Let `ST = Bar.singleton_class` for convenience.*

#### Class variable lookup workflow

Lookup steps for `class_variable_get`:

1. Look up on receiver itself. Go to next step if not found.

2. Look up on `cvar_front_klass`. Front class for `Bar` is `Bar` itself. Front class for `ST` is **the object attached to singleton class**, which is also `Bar`.

3. Look up in ancestors of `cvar_front_klass`, which is `Bar.ancestors`.

From step 2, we can know that:

* In our case `ST.class_variable_get(:@@foo)` is actually equal to `Bar.class_variable_get(:@@foo)`.

* And `ST.singleton_class.class_variable_get(:@@foo)` would look up `@@foo` on ST and ST's ancestors

#### Take a trip into class_variable_get method

Let's find out what's inside `class_variable_get`:

```c
/* file: variable.c */
VALUE
rb_cvar_get(VALUE klass, ID id)
{
    VALUE tmp, front = 0, target = 0;
    st_data_t value;

    tmp = klass;
    CVAR_LOOKUP(&value, {if (!front) front = klass; target = klass;});
    if (!target) {
        rb_name_err_raise("uninitialized class variable %1$s in %2$s",
                          tmp, ID2SYM(id));
    }
    cvar_overtaken(front, target, id);
    return (VALUE)value;
}
```

The core is `CVAR_LOOKUP`, which is defined as:

```c
/* file: variable.c */
#define CVAR_LOOKUP(v,r) do {\
    if (cvar_lookup_at(klass, id, (v))) {r;}\
    CVAR_FOREACH_ANCESTORS(klass, v, r);\
} while(0)
```

It first look up at `klass` itself, and then in each ancestor of `klass`.  The `cvar_lookup_at` is a method to find class variable from klass's instance variables table. Think it as a black box with output of a class variable or 0. Right now it's not important in our short trip. Let's see what's in`CVAR_FOREACH_ANCESTORS`:

```c
/* file: variable.c */
#define CVAR_FOREACH_ANCESTORS(klass, v, r) \
    for (klass = cvar_front_klass(klass); klass; klass = RCLASS_SUPER(klass)) { \
        if (cvar_lookup_at(klass, id, (v))) { \
            r; \
        } \
    }
```

Got `cvar_front_klass` again! So if `klass` here is a singleton class, the `for` loop would **start with instance class of the singleton class**, and then iterate through the ancestors of the instance class. That means, when `ST.class_variable_get(:@@foo)` reaches here, it actually finds `@@foo` in `Bar` and `Bar.ancestors`.

### How ST.singleton_class.class_variable_get(:@@foo) works

*Let `ST = Bar.singleton_class` for convenience.*

#### Lookup steps for `ST.class_variable_get(:@@foo)`

1. `@@foo` is not found on `ST`, then it goes to its front class `Bar`.

2. `Bar` doesn't define `@@foo`. Go to `Bar.ancestors`.

3. Oops! `Bar.ancestors` doesn't contains `Foo` module, so we can never found `@@foo`!

4. Nothing found. End of story.

#### Lookup steps for `ST.singleton_class.class_variable_get(:@@foo)`

1. `@@foo` is not found on `ST.singleton_class`, then it goes to its front class ST.

2. `ST` doesn't define `@@foo`. Go to `ST.ancestors`.

3. `@@foo` is found on `Foo`, one of `ST.ancestors`.

4. Cool! Let's show `'foo'`, the value of `@@foo`!

## Final Words

The key to the mystery of original issue is the collecting and lookup workflows of class variables. Mystery is no mystery once you know how it happens.

At last, the following code works at Ruby 2.7.0:

```ruby
Bar.singleton_class.class_variables
# => []  # <= This is what fixed since 2.7.0

Bar.singleton_class.singleton_class.class_variable_get(:@foo)
# => "foo"  # <= This does not change.
```

(END)
