Posting process
===============

Process of posting and editing messages in Misago is fully modular and extensible.

Whenever there is a need to allow user to post or edit existing message, special pipeline object is initialized and passed `HttpRequest` object istance as well as instances of current user, forum, thread, post and post you are replying to. This pipeline then instiates classes defined in `MISAGO_POSTING_MIDDLEWARES` setting.

This pipeline then asks its middlewares if they have any forms they want to display on posting page, or if they have any actions to perform before, on, or after save to database.

Pipeline itself performs no changes in models or database state, delegating all tasks to its middlewares.

Pipeline is also isolated in database transaction with locks on database rows taking part in posting process. This means that its protected from eventual race conditions and in cause of eventual errors or interruptions, can be rolled back without any downsides.


##### Note

Attachments and any other extra data stored outside of Database that was changed before rollback was called is excluded from this.


## Writing custom middlewares

Middlewares are classes extending `PostingMiddleware` class defined in `misago.threads.posting` package.

During pipeline's initialization each class defined in `MISAGO_POSTING_MIDDLEWARES` setting is initialized and passed current context, which is then stored on its object attributes:

* `mode` - Integer used to identify type of action. You can compare it with `START`, `REPLY` and `EDIT` constants defined in `misago.threads.posting` package.
* `request` - HttpRequest using pipeline.
* `user` - Authenticated user writing message.
* `category` - Category in which message will be posted.
* `thread` - Depending on `mode` and phase in posting process, this may be thread loaded from database, or empty model that is yet to be saved in database.
* `post` - Depending on `mode` and phase in posting process, this may be post loaded from database, or empty model that is yet to be saved in database.
* `quote` - If `mode` is `REPLY`, this may be `None` or instance of message to which reply is written by `user`.
* `datetime` - Instance of `datetime` returned by `django.utils.timezone.now` call. You can use it to avoid subtle differences betwen dates stored on different models generated by new calls to `timezone.now`.
* `parsing_result` - dictionary containing result of parsing message entered by user.

After middleware object has been created, its `use_this_middleware` method is called with no arguments to let middleware make decision if it wants to participate in process and return `True` or `False` accordingly. If you don't define this method yourself, default definition always returns `True`.

In addition to `use_this_middleware` method mentioned ealier, each middleware may define following methods to participate in different phases of posting process:


### `make_form()`

Allows middlewares to display and bind to data their own forms. This function is expected to return None (default behaviour) or Form class instance.

Forms returned by middlewares need to have two special attributes:

* `legend` - Name of form (eg. "Poll").
* `template` - String with path to template that will be used to render this form.

In addition to this form has to declare itself as either main or supporting form via defining `is_main` or `is_supporting` attribute with value being `True`.

If form requires inclusion of JavaScript to work, `js_template` may be defined with path to template to be rendered before document's `</body>`.


### `pre_save(form)`

This method is called with either middleware's form instance or `None` before any state was saved to database.


### `save(form)`

This method is called with either middleware's form instance or `None` to save state to database.


### `post_save(form)`

This method is called with either middleware's form instance or `None` to perform additional actions like sending notifications, etc. ect..


## Reducing number of database saves

Misago provides custom `SaveChangesMiddleware` middleware that allows other middlewares to combine database saves. This middleware is called at the end of `save` and `post_save` phrases, meaning its possible for other middlewares to delegate saves to it during any time of posting process.

This middleware works by defining two extra attributes on `User`, `Forum`, `Thread` and `Post` models: `update_all` and `update_fields`.

If middleware made changes to many fields on model, changing it's `update_all` attribute to `True` will make `SaveChangesMiddleware` call model's `save` method with no arguments at the end of either `save` or `post_save` phase.

However if middleware changed only few fields, it may be better to append their names to model's `update_fields` attribute instead.

When different middlewares add custom fields to `update_fields` and set `update_all` flag at same time, Misago will perform one `save()` call updating whole model.

##### Note

Do not make assumptions or "piggyback" on other middlewares save orders. Introspecting either `update_all` or `update_fields` is bad practice. If your middlware wants its changes saved to database, it should add changed fields names to `update_fields` list even if it already contains them, or set `update_all` to `True`.


## Interrupting posting process from middleware

Middlewares can always interrupt (and rollback) posting process during `interrupt_posting` phrase by raising `misago.threads.posting.PostingInterrupt` exception with error message as its only argument.

All `PostingInterrupt`s raised outside that phase will be escalated to `ValueError` that will result in 500 error response from Misago. However as this will happen inside database transaction, there is chance that no data loss has occured in the process.


## Performing custom validation

Misago defines custom validation framework for posts very much alike one used for [validating new registrations](./ValidatingRegistrations.md).

This framework works by passing user post, title, its parsing result as well as posting middleware's context trough list of callables imported from paths specified in the `MISAGO_POST_VALIDATORS` settings.

Each serializer is expected to be callable accepting two arguments:

* `data` dict with cleaned data. This will `post` containing raw input entered by user into editor, `parsing_result`, an dict defining `parsed_text` key containing parsed message, `mentions` with list of mentions, `images` list of urls to images and two lists: `outgoing_links` and `internal_links`. In case of user posting new thread, this dict will also contain `title` key containing cleaned title.
* `context` dict with context that was passed to posting middleware.

Your validator should raise `from rest_framework.serializers.ValidationError` on error. If validation passes it may return nothing, or updated `data` dict, which allows validators to perform last-minute cleanups on user input.