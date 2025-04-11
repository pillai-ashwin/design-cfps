# CFP-38597: Enhanced L7 Header Matching with Regex Support

**SIG: SIG-Policy** ([View all current SIGs](https://docs.cilium.io/en/stable/community/community/#all-sigs))

**Begin Design Discussion:** 2024-04-11

**Cilium Release:** ?

**Authors:** Ashwin Pillai <pillaiashwin96@gmail.com>

**Status:** Draft

## Summary

Add support for regex-based header matching in Cilium's L7 policy HeaderMatch API. This enhancement allows for more flexible and powerful header matching capabilities in network policies.

## Motivation

Currently, Cilium's L7 policy HeaderMatch API only supports exact value matching for headers. This limitation makes it difficult to implement more complex header matching patterns that could be better expressed using regular expressions. Users often need to match headers against patterns rather than exact values, such as:

- Matching headers with version numbers, path (e.g., `v1.*`)
- Validating header formats
- Supporting wildcard matches in header values

The current workaround would be to create multiple policies with exact matches, which is cumbersome and hard to maintain.

## Goals

* Add regex-based header matching support to the HeaderMatch API
* Maintain backward compatibility with existing exact match behavior
* Provide a clean API that integrates well with Envoy's StringMatcher
* Support safe regular expressions to prevent potential DoS attacks

## Non-Goals

* Change the behavior of existing exact match functionality
* Support other types of pattern matching (prefix, suffix, etc.)
* Modify how header matching works with secrets
* Change how mismatch actions work

## Proposal

### Overview

We propose adding a new `Regex` field to the `HeaderMatch` struct that allows specifying a regular expression pattern for header matching. When specified, this field takes precedence over the `Value` field.

### API Changes

The `HeaderMatch` struct will be updated to include the new `Regex` field:
In `pkg/policy/api/http.go`
```go
type HeaderMatch struct {
    // ... existing fields ...

    // Regex matches the header value using a regular expression.
    // If specified, it overrides the `Value` field.
    //
    // +kubebuilder:validation:Optional
    Regex string `json:"regex,omitempty"`
}
```

### Implementation Details

1. **String Matcher Creation**:
In `pkg/envoy/policy/envoy_l7_rules_translator.go`
   Update the `createStringMatcher` function to handle both exact and regex matches:
   ```go
   func createStringMatcher(value, regex string) *envoy_type_matcher.StringMatcher {
       if regex != "" {
           return &envoy_type_matcher.StringMatcher{
               MatchPattern: &envoy_type_matcher.StringMatcher_SafeRegex{
                   SafeRegex: &envoy_type_matcher.RegexMatcher{
                       Regex: regex,
                   },
               },
           }
       }
       return &envoy_type_matcher.StringMatcher{
           MatchPattern: &envoy_type_matcher.StringMatcher_Exact{
               Exact: value,
           },
       }
   }
   ```

2. **Policy Translation**:
In `pkg/envoy/policy/envoy_l7_rules_translator.go`
   Modify the header matching logic in `getHTTPRule` to use the new regex field:
   ```go
   for _, hdr := range h.HeaderMatches {
       headers = append(headers, &envoy_config_route.HeaderMatcher{
           Name: hdr.Name,
           HeaderMatchSpecifier: &envoy_config_route.HeaderMatcher_StringMatch{
               StringMatch: createStringMatcher(hdr.Value, hdr.Regex),
           },
       })
   }
   ```
3. Write tests in `pkg/envoy/policy/envoy_l7_rules_translator_test.go`

### Example Usage

```yaml
apiVersion: "cilium.io/v2"
kind: CiliumNetworkPolicy
metadata:
  name: "l7-header-regex"
spec:
  endpointSelector:
    matchLabels:
      app: myapp
  ingress:
  - toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          headers:
          - name: "X-API-Version"
            regex: "^v[0-9]+\\.[0-9]+\\.[0-9]+$"
```

## Impacts / Key Questions

### Impact: Performance

Regular expression matching is more computationally expensive than exact matching. However, this impact is mitigated by:
1. Using Envoy's optimized regex engine
2. The optional nature of the feature - users can continue using exact matches
3. Regular expressions being evaluated only for L7 policies

### Impact: Security

Regular expressions could potentially be used for DoS attacks. To mitigate this:
1. Use Envoy's safe regex engine which has built-in protections
2. Consider adding regex complexity limits in future iterations

### Key Question: Precedence Rules

How should we handle cases where both `Value` and `Regex` fields are specified?

### Option 1 (Selected):

Regex takes precedence over Value field

#### Pros
* Clear precedence rules
* Consistent with other systems where more specific matches take precedence
* Easy to document and understand

#### Cons
* Might be unexpected for some users
* Requires careful documentation

### Option 2:

Return an error if both fields are specified

#### Pros
* Forces users to be explicit about their intent
* Prevents ambiguity

#### Cons
* Less flexible
* Makes migration more difficult
* Breaks backward compatibility if users have both fields set

## Future Milestones

### Support for Additional Match Types

Consider adding support for other match types supported by Envoy's StringMatcher:
* Prefix matching
* Suffix matching
* Contains matching

### Regex Complexity Limits

Add configuration options to limit regex complexity to prevent DoS attacks:
* Maximum regex length
* Maximum backtracking steps
* Regex syntax restrictions
