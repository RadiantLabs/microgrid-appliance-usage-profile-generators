## Microgrid Appliance Usage Profile Generators

Scripts to convert measured appliance usage into yearly usage profiles. Used for Mills, welders, water pumps and similar appliances

#### What is kw_factor?

When the motor is running at full RPMs for 1 minute, we have a kw_factor of 1 (of course, it’s really 2 minute intervals, so full RPM for 2 minutes is 2 kw_factor).

#### Approach to convert RPM to unitless usage counts
RPMs relate to 3 quantities that we care about:
* kW
* kWh
* Grain throughput

These quantities are directly related to costs and revenue for the appliance owner and grid operator. We have measured revolution count in 2-minute intervals. The average RPM is half the number of mill revolutions.

However, if we want to work with hourly intervals, then we can't add up RPMs to get the total RPMs within an hour. We have a few options:
1. Average the RPMs across the hour. If the mill runs for 30 minutes at 1200 RPMs, then the average would be 600 RPMs. Grain throughput and kW likely have a non-linear relationship to RPMs, so this would really distort the results.
2. Assign a unitless usage factor to each minute interval. If the mill is running at full production (throughput and kW), then we can assign it a value of 1 (2 for a 2-minute interval). We can scale the factor down as the RPMs go down.
The second approach is what we use.

#### Throughput vs. kW
To calculate kW (and therefore kWh) and throughput, we need to know how RPM relates to these two factors. There is no reason to assume the kW and grain throughput vs RPM scale the same. So let's create two functions that calculate kW and grain throughput independently.
For now, these derate factors will be the same. But the functions can be easily modified as we get more information without refactoring code downstream.

#### RPM to unitless utilization factors
This rice mill is likely rated at 16.2kW @ 2200 RPM. This one looks like it runs around 5000 RPM. Either way, let's assume some function that relates it's utilization given an RPM.

```
# What we have measured from the mill sensors is revolutions per 2 minutes: `rp2m`. (At this point we've already converted to rpm by dividing it by 2)
# What we want is the percent that this mill is fully utilized over the course of an hour: `kw_factor`.
# This will allow us to multiply this factor by the appliance power (nominal power in kW) to get the kW for that hour.
# This should stay true as long as kW scales linearly with RPM. We can refine later as we get more data.

# We may want to filter very low RPMs out. A slow motor still uses power and mills grain, but I don't know the cutoff
# Assuming we have already
def rpm_to_kw_utilization(rpm, full_capacity_rpm):
    """
    Convert RPM to a kW utilization factor.
    This assumes we have already converted the 2-minute revolution count (rp2m) count to 1 minute (rpm)
    """
    # Fraction of full capacity over 2 minutes
    two_min_rpm_full_capacity_fraction = rpm / full_capacity_rpm

    # There are 30 2-minute intervals in an hour
    intervals_per_hour_count = 30

    # Calculate the fraction of full utilization for the *entire hour* this 2 minute interval
    # contributes. Then when we resample fromm 2min to 1 hour intervals, these will sum correctly.
    # This is a little unintuitive - we could divide by 30 after resampling, but then this
    # calculation is spread out across this notebook and may be more confusing
    return round(two_min_rpm_full_capacity_fraction / intervals_per_hour_count, 5)

rpm_to_kw_utilization(6200, full_capacity_rpm)
```
