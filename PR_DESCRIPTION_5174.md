# [config] Add explicit exchanger configuration support for generic references
## Related issue
- Closes #5174

## Problem
Issue #5174 asks how to configure an `exchanger` extension for Dubbo generic invocation.  
On the provider/protocol side, `exchanger` is already a first-class configuration item, but on the consumer/reference side it was not exposed consistently across public config surfaces.  
Users could still pass it indirectly via generic `parameters`, but there was no clear, explicit, and discoverable `exchanger` option for reference/generic usage in:
- reference/consumer config objects,
- `@Reference` / `@DubboReference` annotations,
- builder APIs,
- and Spring XSD attributes.

## Solution
Expose `exchanger` as a first-class reference-side option and wire it through all major configuration entry points.

### Code changes
- Added `exchanger` field with getter/setter in:
  - `AbstractReferenceConfig`
- Added `exchanger` annotation attribute in:
  - `org.apache.dubbo.config.annotation.Reference`
  - `org.apache.dubbo.config.annotation.DubboReference`
  - `com.alibaba.dubbo.config.annotation.Reference`
- Added builder support in:
  - `AbstractReferenceBuilder#exchanger(String)`
  - `ReferenceBeanBuilder#setExchanger(String)`
- Added spring reference key constant:
  - `ReferenceAttributes.EXCHANGER`
- Added XSD support for consumer/reference attributes in:
  - `META-INF/dubbo.xsd`
  - `META-INF/compat/dubbo.xsd`
- Updated compatibility definition snapshots for exposed accessors in:
  - `definition/com.alibaba.dubbo.config.ConsumerConfig`
  - `definition/com.alibaba.dubbo.config.ReferenceConfig`

### Test updates
- Added/updated focused tests:
  - `AbstractReferenceConfigTest#testExchanger`
  - `AbstractReferenceBuilderTest#exchanger` and `build` assertion
  - `ConsumerBuilderTest#exchanger` and `build` assertion

## Why this addresses #5174
Generic invocation uses the same reference-side config pipeline as normal references.  
By making `exchanger` explicitly configurable on reference surfaces, users can directly configure exchanger extensions for generic calls without relying on opaque parameter maps.

## Backward compatibility
- Backward compatible:
  - Existing behavior is preserved.
  - Existing parameter-based configurations continue to work.
  - This change only adds explicit configuration paths.

## Verification
Executed:

```bash
/home/calelin/dev/dubbo/mvnw -f /home/calelin/dev/dubbo/pom.xml -pl dubbo-common spotless:apply
/home/calelin/dev/dubbo/mvnw -f /home/calelin/dev/dubbo/pom.xml -pl dubbo-config/dubbo-config-api -am -Dtest=AbstractReferenceConfigTest,AbstractReferenceBuilderTest,ConsumerBuilderTest -DfailIfNoTests=false -Dsurefire.failIfNoSpecifiedTests=false test
```

Result:
- Build success
- Focused tests passed
