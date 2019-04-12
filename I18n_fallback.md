

(copypasta from "Sven's blog post":http://svenfuchs.com/2009/7/19/experimental-ruby-i18n-extensions-pluralization-fallbacks-gettext-cache-and-chained-backend)

Another feature that was requested quite often, too, is Locale fallbacks. Simple backend just returns a “translation missing” error message or raises an exception if you tell it so. It won’t check any other locales if it can’t find a translation for the current or given locale though.

There were proposals for a minimal fallback functionality that just checks the default locale’s translations if a translation is not available for the current locale. Globalize2 on the other hand ships with a quite powerful Locale fallbacks implementation that also enforces RFC 4646/47 standard compliant locale (language) tags.

I’ve discussed this with Joshua and we’ve decided to extract a simplified version from Globalize2’s fallbacks that makes the RFC 4646 standard compliance an optional feature but still allows enough flexibility to define arbitrary fallback rules if you need them. If you don’t define anything it will just use the default locale as a single fallback locale.

Again enabling Locale fallbacks is just a matter of including the module to any compatible backend:

bc. require "i18n/backend/fallbacks" 
I18n::Backend::Simple.send(:include, I18n::Backend::Fallbacks)

This overwrites the Base backend’s translate method so that it will try each locale given by I18n.fallbacks for the given locale. E.g. for the locale :"de-DE" it will try the locales :"de-DE", :de and :en until it finds a result with the given options. If it does not find any result for any of the locales it will then raise a MissingTranslationData exception as usual.

h2. Providing A Default

The :default option will provide a default translation either before or after falling back to another locale based on it's class. Given the following translations:

bc. I18n.backend.store_translations(:en, :foo => 'Foo in :en', :bar => 'Bar in :en')
I18n.backend.store_translations(:de, :bar => 'Bar in :de')

If the :default option is a Symbol, it will be looked up for the given locale before falling back to another locale:

bc. I18n.translate(:foo, :locale => :de, :default => :bar)         # => 'Bar in :de'
I18n.translate(:foo, :locale => :de, :default => :missing_foo) # => 'Foo in :en'

When the :default option is a String it is evaluated last after all the fallback locales have been tried:

bc. I18n.translate(:foo, :locale => :de, :default => 'Default Foo')         # => 'Foo in :en'
I18n.translate(:missing_foo, :locale => :de, :default => 'Default Foo') # => 'Default Foo'

And similarly for a Proc:

bc. I18n.translate(:foo, :locale => :de, :default => Proc.new {'Default Foo'})         # => 'Foo in :en'
I18n.translate(:missing_foo, :locale => :de, :default => Proc.new {'Default Foo'}) # => 'Default Foo'

You can skip the fallback and return the default by passing in `false`.

bc. I18n.translate(:foo, :locale => :de, :default => 'Default Foo', :fallback => false)   # => 'Default Foo'

h2. Custom Fallback Rules

You can add custom fallback rules to the I18n.fallbacks instance like this:

bc. # use Spanish translations if Catalan translations are missing:
I18n.fallbacks.map(:ca => :"es-ES")
I18n.fallbacks[:ca] # => [:ca, :"es-ES", :es, :en]

If you do not add any custom fallback rules it will just use the default locale and the default locales fallbacks:

bc. # using :"en-US" as a default locale:
I18n.default_locale = :"en-US" 
I18n.fallbacks[:ca] # => [:ca, :"en-US", :en]

h2. RFC 4646 standard compliance

If you want RFC 4646 standard compliance to be enforced for your locales you can use the Rfc4646 Tag class:

bc. I18n::Locale::Tag.implementation = I18n::Locale::Tag::Rfc4646

This will make a locale “de-Latn-DE-1996-a-ext-x-phonebk-i-klingon” fall back to the following locales in this order:

bc. de-Latn-DE-1996-a-ext-x-phonebk-i-klingon
de-Latn-DE-1996-a-ext-x-phonebk
de-Latn-DE-1996-a-ext
de-Latn-DE-1996
de-Latn-DE
de-Latn
de

p. Most of the time you probably won’t need anything like this. Thus we’ve used the (much cheaper) I18n::Locale::Tag::Simple class as the default implementation. It simply splits locales at dashes and thus can do fallbacks like this:

bc. de-Latn-DE-1996
de-Latn-DE
de-Latn
de

p. Should be good enough in most cases, right :)
