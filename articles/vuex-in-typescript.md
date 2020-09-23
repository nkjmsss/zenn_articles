---
title: "VuexをTypeScriptでいい感じに書く"
emoji: ":sparkles:"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "vue", "vuex"]
published: true
---

## はじめに

`commit` や `dispatch` の第一引数が `string` のため，型情報が完全に失われてしまう．それにより，TypeScriptで書いていても型安全にすることはできず，書き心地がよくない．

本記事では， `commit` 等のラッパーを作ることで，型安全にそれらを使えるようにする方法を示す．

### 他の解決法

例えば，[vuex-module-decorators](https://github.com/championswimmer/vuex-module-decorators)というライブラリがある．しかし，nuxtでは当該ライブラリを使うにはclassic modeを使うしかなく，deprecatedである．また，decorator記法もexperimentalな機能のためプロジェクトによっては好まれないことも多い．

## 自分で型情報を作る

```typescript
// state
import { ActionTree, Commit, CommitOptions, Dispatch, DispatchOptions, MutationTree, Store } from 'vuex'

const moduleName = 'sample'

// state
interface State {
  count: number
}

export const state = (): State => ({
  count: 0,
})

// mutations
const makeMutations = <T extends MutationTree<State>>(mutations: T): T => mutations

export const mutations = makeMutations({
  increment(state) {
    state.count++
  },

  incrementBy(state, payload: number) {
    state.count += payload
  },
})

// actions
const makeActions = <T extends ActionTree<State, unknown>>(actions: T): T => actions

export const actions = makeActions({
  increment({ commit }): void {
    _commit(commit, 'increment')
  },

  async incrementAsync({ commit }): Promise<number> {
    // mimic api
    const sleep = (t: number): Promise<number> => new Promise((resolve, reject) => setTimeout(() => resolve(t), t))

    const res = await sleep(1000)
    _commit(commit, 'incrementBy', res)
    return res
  },

  dispatchOtherAction({ dispatch }): void {
    _dispatch(dispatch, 'incrementAsync').then(v => {
      console.log(v)
    })
  },
})

// util-commit
type CommitPayload<K extends keyof typeof mutations> = Parameters<typeof mutations[K]>[1]
type CommitParameters<K extends keyof typeof mutations> = CommitPayload<K> extends undefined
  ? [K] | [K, CommitPayload<K>] | [K, CommitPayload<K>, CommitOptions]
  : [K, CommitPayload<K>] | [K, CommitPayload<K>, CommitOptions]

const _commit = <K extends keyof typeof mutations>(
  commit: Commit,
  ...[key, payload, options]: CommitParameters<K>
): void => commit(key, payload, options)

export const commit = <K extends keyof typeof mutations>(
  store: Store<any>,
  ...[key, payload]: CommitParameters<K>
): void => store.commit(`${moduleName}/${key}`, payload)

// util-actions
type DispatchPayload<K extends keyof typeof actions> = Parameters<typeof actions[K]>[1]
type DispatchParameters<K extends keyof typeof actions> = DispatchPayload<K> extends undefined
  ? [K] | [K, DispatchPayload<K>] | [K, DispatchPayload<K>, DispatchOptions]
  : [K, DispatchPayload<K>] | [K, DispatchPayload<K>, DispatchOptions]

const _dispatch = <K extends keyof typeof actions>(
  dispatch: Dispatch,
  ...[key, payload, options]: DispatchParameters<K>
): ReturnType<typeof actions[K]> => dispatch(`${moduleName}/${key}`, payload, options) as ReturnType<typeof actions[K]>

export const dispatch = <K extends keyof typeof actions>(
  store: Store<any>,
  ...[key, payload, options]: DispatchParameters<K>
): ReturnType<typeof actions[K]> =>
  store.dispatch(`${moduleName}/${key}`, payload, options) as ReturnType<typeof actions[K]>

```

まず， `MutationTree` や `ActionTree` はkeyがstringとされているため，何もしない関数を通してkeyの情報を取り出せるようにしています（より良い方法があれば知りたいです）．

それを利用して， `dispatch` や `commit` のラッパー関数を作ることで， `commit` の引数が `string` でしかないことを解消し，さらに `payload` にも型を与えられるようにしています．また， `payload` が必要かどうかも含めた型になっているので， `incrementBy` を `commit` する際には `payload` を与えないと型エラーになるようになっています．

コンポーネント側での利用も同様に型情報付きで行えるようにしており，以下のように書くことができます．

```typescript
import Vue from 'vue'
import * as SampleStore from '~/store/sample'

export default Vue.extend({
  mounted() {
    SampleStore.dispatch(this.$store, 'increment')
  }
})
```
