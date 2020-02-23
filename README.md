# 🔑 MSIG for `setinflation` to 1%

Reference document for modifying inflation on an EOSIO chain.

- **Proposer**: `eosnationftw`
- **Proposal Name**: `setinflation`

**Proposed on following Testnets:**

- [Kylin](https://kylin.bloks.io/msig/eosnationftw/setinflation)
- [Jungle2](https://jungle.bloks.io/msig/eosnationftw/setinflation)
- [Jungle3](https://monitor3.jungletestnet.io/)

## Proposed Parameters

The following `setinflation` parameters would set the total inflation to 1% and nothing would flow into `eosio.saving`.

- Annual rate: `100` (1%)
- Inflation pay factor: `10000` (100%)
- Vote pay factor: `40000` (25%)

## Action

JSON file for [setinflation.json](setinflation.json)

```bash
cleos push action eosio setinflation '{"annual_rate":100, "inflation_pay_factor":10000, "votepay_factor":40000}' -p eosio
```

# EOSIO code reference

## Default Parameters

- Annual rate: `500` (5%)
- Inflation pay factor: `50000` (20%)
- Vote pay factor: `40000` (25%)

[eosio.system.hpp#L73-L77](https://github.com/EOSIO/eosio.contracts/blob/7109f002fa9afff18edcafce5c32b224a21eef98/contracts/eosio.system/include/eosio.system/eosio.system.hpp#L73-L77)

```c++
static constexpr int64_t  inflation_precision           = 100;     // 2 decimals
static constexpr int64_t  default_annual_rate           = 500;     // 5% annual rate
static constexpr int64_t  pay_factor_precision          = 10000;
static constexpr int64_t  default_inflation_pay_factor  = 50000;   // producers pay share = 10000 / 50000 = 20% of the inflation
static constexpr int64_t  default_votepay_factor        = 40000;   // per-block pay share = 10000 / 40000 = 25% of the producer pay
```

## Producer Pay

- **To Producers** = `new_tokens` * `pay_factor_precision` / `inflation_pay_factor`;
- **To Savings** = `new_tokens` - `to_producers`

[producer_pay.cpp#L85-L93](https://github.com/EOSIO/eosio.contracts/blob/7109f002fa9afff18edcafce5c32b224a21eef98/contracts/eosio.system/src/producer_pay.cpp#L85-L93)

```c++
double additional_inflation = (_gstate4.continuous_rate * double(token_supply.amount) * double(usecs_since_last_fill)) / double(useconds_per_year);
check( additional_inflation <= double(std::numeric_limits<int64_t>::max() - ((1ll << 10) - 1)),
    "overflow in calculating new tokens to be issued; inflation rate is too high" );
int64_t new_tokens = (additional_inflation < 0.0) ? 0 : static_cast<int64_t>(additional_inflation);

int64_t to_producers     = (new_tokens * uint128_t(pay_factor_precision)) / _gstate4.inflation_pay_factor;
int64_t to_savings       = new_tokens - to_producers;
int64_t to_per_block_pay = (to_producers * uint128_t(pay_factor_precision)) / _gstate4.votepay_factor;
int64_t to_per_vote_pay  = to_producers - to_per_block_pay;
```

## Set Inflation

Checks if `inflation_pay_factor` & `votepay_factor` is lower than 10000 (100%).

[eosio.system.cpp#L298-L311](https://github.com/EOSIO/eosio.contracts/blob/v1.8.3/contracts/eosio.system/src/eosio.system.cpp#L298-L311)

```c++
void system_contract::setinflation( int64_t annual_rate, int64_t inflation_pay_factor, int64_t votepay_factor ) {
    require_auth(get_self());
    check(annual_rate >= 0, "annual_rate can't be negative");
    if ( inflation_pay_factor < pay_factor_precision ) {
        check( false, "inflation_pay_factor must not be less than " + std::to_string(pay_factor_precision) );
    }
    if ( votepay_factor < pay_factor_precision ) {
        check( false, "votepay_factor must not be less than " + std::to_string(pay_factor_precision) );
    }
    _gstate4.continuous_rate      = get_continuous_rate(annual_rate);
    _gstate4.inflation_pay_factor = inflation_pay_factor;
    _gstate4.votepay_factor       = votepay_factor;
    _global4.set( _gstate4, get_self() );
}
```

## Get Continous Rate

- 5% inflation = 0.04879016416943201
- 1% inflation = 0.009950330853168083

```js
> Math.log1p(500 / (100 * 100))
0.04879016416943201
> Math.log1p(100 / (100 * 100))
0.009950330853168083
```

[eosio.system.cpp#L15](https://github.com/EOSIO/eosio.contracts/blob/v1.8.3/contracts/eosio.system/src/eosio.system.cpp#L15)

```c++
double get_continuous_rate(int64_t annual_rate) {
    return std::log1p(double(annual_rate)/double(100*inflation_precision));
}
```
