
[API Tool](https://github.com/subor/sdk/blob/master/docs/topics/build_sdk_source.md#thrift) to deal with:  
![]({{ "/assets/devtool_apitool_rs.png" | absolute_url }})

```
18/08/22 12:54:53  Info ApiTool --ThriftFiles="D:\dev\jade\jade\sdk\ThriftFiles" --ThriftExe="D:\dev\jade\jade\..\tools\thrift\thrift.exe" --CommonOutput="D:\dev\jade\jade\sdk\SDK.Gen.CommonAsync" --ServiceOutput="D:\dev\jade\jade\sdk\SDK.Gen.ServiceAsync" --Gen="rs" --Generate
18/08/22 12:54:53  Info -gen rs -out D:\dev\jade\jade\sdk\SDK.Gen.ServiceAsync D:\dev\jade\jade\sdk\ThriftFiles\BrainCloudService\BrainCloudServiceSDKDataTypes.thrift
18/08/22 12:54:53  Info -gen rs -out D:\dev\jade\jade\sdk\SDK.Gen.ServiceAsync D:\dev\jade\jade\sdk\ThriftFiles\BrainCloudService\BrainCloudServiceSDKServices.thrift
18/08/22 12:54:54  Info -gen rs -out D:\dev\jade\jade\sdk\SDK.Gen.CommonAsync D:\dev\jade\jade\sdk\ThriftFiles\CommonType\CommonTypeSDKDataTypes.thrift
```

```
cargo new --lib subor
```

I found `localization_service_s_d_k_data_types.rs`:
```rust
impl LanguageChangedMsg {
  pub fn new<F1, F2>(new_language: F1, old_language: F2) -> LanguageChangedMsg where F1: Into<Option<String>>, F2: Into<Option<String>> {
    LanguageChangedMsg {
      new_language: new_language.into(),
      old_language: old_language.into(),
    }
  }
  //...
```

To the top of lib.rs add:
```rust
mod localization_service_s_d_k_data_types;
```

If you're rocking [Visual Studio Code](https://code.visualstudio.com/) bring up the terminal with ```^` ``` (that's Ctrl+Backtick- or "grave accent" if you're fancy).  Probably also want [Rust support](https://marketplace.visualstudio.com/items?itemName=rust-lang.rust).

Build with `cargo build`:
```
error[E0463]: can't find crate for `ordered_float`
 --> src/localization_service_s_d_k_data_types.rs:9:1
  |
9 | extern crate ordered_float;
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^ can't find crate
```

To Cargo.ml add:
```
[dependencies]
ordered-float = "0.5.0"
thrift = "0.0.4"
try_from = "0.2.2"
```

Build:
```
error[E0432]: unresolved import `ordered_float`
  --> src/localization_service_s_d_k_data_types.rs:13:5
   |
13 | use ordered_float::OrderedFloat;
   |     ^^^^^^^^^^^^^ Did you mean `self::ordered_float`?
```

To the top of lib.rs add:
```rust
extern crate ordered_float;
extern crate thrift;
extern crate try_from;
```

```rust
#[cfg(test)]
mod tests {
    use super::localization_service_s_d_k_data_types::*;
    #[test]
    fn it_works() {
```

Inside `it_works()` function type `let msg = L` (Note: capital __"L"__) and auto-complete should work:
```rust
let msg = LanguageChangedMsg::new("stuff".to_owned(), "this".to_owned());
```