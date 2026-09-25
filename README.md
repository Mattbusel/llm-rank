# llm-rank

[![CI](https://github.com/Mattbusel/llm-rank/actions/workflows/ci.yml/badge.svg)](https://github.com/Mattbusel/llm-rank/actions/workflows/ci.yml)
![C++17](https://img.shields.io/badge/C%2B%2B-17-blue.svg)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

**Rerank search results by relevance: offline BM25, LLM scoring, or both.** One header, `llm_rank.hpp`. Needs libcurl (`apt install libcurl4-openssl-dev`, preinstalled on macOS, `vcpkg install curl` on Windows).

The first pass of retrieval (keyword search or vector similarity) is fast but noisy. Reranking the top results before they go into a prompt improves answers and saves tokens. llm-rank gives you a classic BM25 ranker that runs locally, an LLM judge that scores each passage 0 to 10, and a hybrid that uses the cheap one to shortlist for the expensive one.

## Features

- `rerank_local()`: BM25 blended with keyword overlap, tunable weights, `k1` and `b`; no network
- `rerank_llm()`: asks a chat model to rate each passage with a customisable scoring prompt
- `rerank_hybrid()`: BM25 over everything, then LLM scoring on the top `llm_top_k`
- `BM25` class for raw scores over your own corpus
- Results carry the score and both the original and the new rank

## Quick start

Copy the header into your project:

```bash
curl -fsSLO https://raw.githubusercontent.com/Mattbusel/llm-rank/main/include/llm_rank.hpp
```

Define `LLM_RANK_IMPLEMENTATION` in exactly one `.cpp` file before including it; every other file just includes the header. Save this as `main.cpp` next to the header:

```cpp
#define LLM_RANK_IMPLEMENTATION
#include "llm_rank.hpp"
#include <cstdio>

int main() {
    std::vector<std::string> passages = {
        "Gradient descent is a first-order optimization algorithm.",
        "The Eiffel Tower is in Paris.",
        "Adam adapts the learning rate for each parameter.",
    };

    // Offline BM25 + keyword scoring: no API key, no network.
    for (const auto& r : llm::rerank_local("learning rate optimization", passages))
        std::printf("#%d (was #%d)  %.3f  %s\n",
                    r.new_rank, r.original_rank, r.score, r.passage.c_str());

    // Raw BM25 scores over a corpus.
    llm::BM25 bm25(passages);
    auto scores = bm25.scores("optimization");
    std::printf("bm25[0] = %.3f\n", scores[0]);
}
```

```bash
g++ -std=c++17 -O2 main.cpp -lcurl -o demo
```

## API at a glance

| Call | Purpose |
|---|---|
| `rerank_local(query, passages, LocalRankConfig)` | Offline BM25 + keyword ranking |
| `rerank_llm(query, passages, LLMRankConfig)` | Model-scored ranking (OpenAI chat completions) |
| `rerank_hybrid(query, passages, local_cfg, llm_cfg, llm_top_k)` | Shortlist locally, then score with the model |
| `BM25(corpus, k1, b).scores(query)` | Raw BM25 scores |

## Notes and limitations

- The implementation includes `<curl/curl.h>`, so you need libcurl headers and `-lcurl` even if you only use the BM25 functions.
- LLM scoring makes one API call per passage; keep the candidate list short or use the hybrid mode.

## Build the examples

The repo builds `examples/basic_rerank.cpp`, `examples/bm25_demo.cpp`, `examples/llm_rerank.cpp`, `examples/hybrid_rerank.cpp` with CMake (requires libcurl):

```bash
cmake -B build
cmake --build build
```

## Part of llm-cpp

llm-rank is one of 26 single-header C++ libraries in [llm-cpp](https://github.com/Mattbusel/llm-cpp), a toolkit for building LLM features into native code. Each library stands alone; combine them by giving each `*_IMPLEMENTATION` define its own `.cpp` file. See the [llm-cpp README](https://github.com/Mattbusel/llm-cpp#using-several-together) for the full list and examples of using several together.

## License

MIT. See [LICENSE](LICENSE).
