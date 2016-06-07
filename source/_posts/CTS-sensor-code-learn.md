---
title: CTS sensor code learn
date: 2016-06-07 19:54:25
tags:
---
# 0x00 前言
----
Android设备生产厂商众多，系统ROM个性化定制也很多。为了保证Android的兼容性，Google推出了一个测试工具CTS。
>The Compatibility Test Suite (CTS) is a free, commercial-grade test suite, 》available for download. The CTS represents the "mechanism" of compatibility.

CTS中还有一部分需要手动测试，叫做CTS Verifier。
>The Compatibility Test Suite Verifier (CTS Verifier) is a supplement to the CTS available for download. CTS Verifier provides tests for APIs and functions that cannot be tested on a stationary device without manual input (e.g. audio quality, accelerometer, etc).

只有通过了CTS认证的手机可以使用Google Play，可以提起GMS认证。
通过了Google Mobile Service（GMS）认证，出货的时候就可以内置GMS服务。

要通过GMS认证 ，需要另一个工具XTS。

一般情况下，我们说CTS认证，都是会跑CTS/CTS Verifier/XTS， google审查了之后就可以在[Supported devices](https://support.google.com/googleplay/android-developer/answer/6154891?hl=zh-Hant&rd=1)的清单中查看到。

# 0x01 sensor相关类
----

![](/imgs/CTS-Sensor-code-learn/classes.png)


TestSensorOperation是一个核心类，所有的测试代码都会调用execute方法。在该方法中，会调用接口Executor的具体实现者的execute方法。这个方法一般都是注册sensor的监听事件，由TestSensorManager实现。

事件由TestSensorEventListener收集。利用CountDownLatch来实现线程等待，等待需要的事件收集完成。如果在一段时间里没有收集完成，就会提示失败。

数据收集完成，就会由接口ISensorVerification的实现者做各种检查。

常见的如：
EventGapVerification、EventOrderingVerification、FrequencyVerification

# 0x02 举例说明
----
在android-cts-verifier-6.0_r6版本中，新增加了测试项testTimestampClockSource
![](/imgs/CTS-Sensor-code-learn/timestampClockSourceCommit.jpg)

主要关注TimestampClockSourceVerification。

__public class TimestampClockSourceVerification extends AbstractSensorVerification__
__public abstract class AbstractSensorVerification implements ISensorVerification__

下面就直接上代码了
DeviceSuspendTestActivity.java
```java
public String testTimestampClockSource() throws Throwable {
String string = null;

......

for (Sensor sensor : sensorList) {
    if (sensor.getReportingMode() == Sensor.REPORTING_MODE_CONTINUOUS) {
        try {
            string = runVerifySensorTimestampClockbase(sensor, false);
            if (string != null) {
                return string;
            }
        } catch(Throwable e) {
            Log.e(TAG, e.getMessage());
            error_occurred = true;
        }
    } else {
        Log.i(TAG, "testTimestampClockSource skipping non-continuous sensor: '" + sensor.getName());
    }
}
if (error_occurred) {
    throw new Error("Sensors must use CLOCK_BOOTTIME as clock source for timestamping events");
}
return null;
}

public String runVerifySensorTimestampClockbase(Sensor sensor, boolean verify_clock_delta) throws Throwable {
    Log.i(TAG, "Running .. " + getCurrentTestNode().getName() + " " + sensor.getName());
    if (verify_clock_delta) {
        verifyClockDelta();
    }
    /* Enable a sensor, grab a sample, and then verify timestamp is > realtimeNs
     * to assure the correct clock source is being used for the sensor timestamp.
     */
    final int MIN_TIMESTAMP_BASE_SAMPLES = 1;
    int samplingPeriodUs = sensor.getMinDelay();
    TestSensorEnvironment environment = new TestSensorEnvironment(
            this,
            sensor,
            false,
            (int) samplingPeriodUs,
            0,
            false /*isDeviceSuspendTest*/);
    TestSensorOperation op = TestSensorOperation.createOperation(environment, MIN_TIMESTAMP_BASE_SAMPLES);
    op.addVerification(TimestampClockSourceVerification.getDefault(environment));
    try {
        op.execute(getCurrentTestNode());
    } finally {
    }
    return null;
}

```

先实例化TestSensorEnvironment ，接着创建一个TestSensorOperation ，添加自定义的ISensorVerification，最后调用execute开始收集数据。

下面看看文件TimestampClockSourceVerification.java
```java
public class TimestampClockSourceVerification extends AbstractSensorVerification {
...
Override
public void verify(TestSensorEnvironment environment, SensorStats stats) {
    StringBuilder errorMessageBuilder =
            new StringBuilder(" Incorrect timestamp clock source failures: ");
    boolean success = false;
    int failuresCount = 0;
    List<IndexedEvent> failures;

    try {
        failures = verifyTimestampClockSource(errorMessageBuilder);
        failuresCount = failures.size();
        stats.addValue(SensorStats.EVENT_TIME_WRONG_CLOCKSOURCE_COUNT_KEY, failuresCount);
        stats.addValue(
                SensorStats.EVENT_TIME_WRONG_CLOCKSOURCE_POSITIONS_KEY,
                getIndexArray(failures));
        success = failures.isEmpty();
    } catch (Throwable e) {
        failuresCount++;
        stats.addValue(SensorStats.EVENT_TIME_WRONG_CLOCKSOURCE_COUNT_KEY, 0);
    }
    stats.addValue(PASSED_KEY, success);
    errorMessageBuilder.insert(0, failuresCount);
    Assert.assertTrue(errorMessageBuilder.toString(), success);
}


private List<IndexedEvent> verifyTimestampClockSource(StringBuilder builder) throws Throwable {
    int collectedEventsCount = mCollectedEvents.size();
    ArrayList<IndexedEvent> failures = new ArrayList<IndexedEvent>();

    if (collectedEventsCount == 0) {
        if (failures.size() < TRUNCATE_MESSAGE_LENGTH) {
            builder.append("No events received !");
        }
        Assert.assertTrue("No events received !", false);
    }

    for (int i = 0; i < collectedEventsCount; ++i) {
        TestSensorEvent event = mCollectedEvents.get(i);
        long eventTimestampNs = event.timestamp;
        long receivedTimestampNs = event.receivedTimestamp;
        long upperThresholdNs = receivedTimestampNs;
        long lowerThresholdNs = receivedTimestampNs - mMaximumLatencyNs;

        if (eventTimestampNs < lowerThresholdNs || eventTimestampNs > upperThresholdNs) {
            if (failures.size() < TRUNCATE_MESSAGE_LENGTH) {
                builder.append("position=").append(i);
                builder.append(", timestamp=").append(String.format("%.2fms",
                            nanosToMillis(eventTimestampNs)));
                builder.append(", expected=[").append(String.format("%.2fms",
                            nanosToMillis(lowerThresholdNs)));
                builder.append(", ").append(String.format("%.2f]ms; ",
                            nanosToMillis(upperThresholdNs)));
            }
            failures.add(new IndexedEvent(i, event));
        }
    }
    if (failures.size() >= TRUNCATE_MESSAGE_LENGTH) {
        builder.append("more; ");
    }
    return failures;
}

...
}
```
# 0x03 资料
----
https://android.googlesource.com/platform/cts/+/android-cts-6.0_r6/
