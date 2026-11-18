> Published and verified on 2026-11-17:
> https://github.com/juju/juju/pull/23731#issuecomment-6965404634
>
> Destination:
> https://github.com/juju/juju/pull/23731#issuecomment-6953629797
> Action: post a PR conversation comment.

Thanks for checking the QA. I rebased the branch onto current 4.1 and
squashed the intermediate commits into one, keeping the final sequence
implementation unchanged. The sequence package race tests passed on
SQLite and dqlite; PostgreSQL execution was not tested.
