# market-data

Daily market and AI news data in JSON.

## Files

| Path | Contents |
| --- | --- |
| `latest/markets.json` | Newest markets day |
| `latest/ai.json` | Newest AI day |
| `days/YYYY-MM-DD/markets.json` | Markets for that date: summary, story cards, rates, stock notes, calendar, prices, and the afternoon close update when there is one |
| `days/YYYY-MM-DD/ai.json` | AI news cards for that date |
| `index.json` | Available dates and the files for each |

Raw URLs work from any web page (CORS `*`), for example:

```
https://raw.githubusercontent.com/nick-nguyen-ai/market-data/main/latest/markets.json
```

Each story links to its original sources. Prices are delayed and unofficial.
Information only, not financial advice.
