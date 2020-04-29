# reactor-poc

## Example 1: Hot Streams with backpressure
Flux is one of implementations of the Reactive Streams Publisher interface. This example uses `Flux`. It's a stream that can emit 0..n elements continously. In real life cases stream as it name suggests flows infinitely. Hot streams are always running and can be subscribed to at any point in time. One way to create a hot stream is by converting a cold stream into one. By calling `publish()` we are given a `ConnectableFlux`. Calling `subscribe()` will not trigger emitting events, it allows adding multiple subscriptions. Once we call `connect()`, that the Flux will start emitting. It doesn't matter whether we are subscribing or not. Backpressure is when a downstream can tell an upstream to send it fewer data in order to prevent it from being overwhelmed. See how onNext method slows down the event emitting.

```java



import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import org.reactivestreams.Subscriber;
import org.reactivestreams.Subscription;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import reactor.core.publisher.ConnectableFlux;
import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;

public class Streaming {
  static Logger logger = LoggerFactory.getLogger(Streaming.class);
  static String AlphaNumericString = "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
      + "0123456789"
      + "abcdefghijklmnopqrstuvxyz";

  static String getAlphaNumericString(int n)
  {

    // chose a Character random from this String

    // create StringBuffer size of AlphaNumericString
    StringBuilder sb = new StringBuilder(n);

    for (int i = 0; i < n; i++) {

      // generate a random number between
      // 0 to AlphaNumericString variable length
      int index
          = (int)(AlphaNumericString.length()
          * Math.random());

      // add Character one by one in end of sb
      sb.append(AlphaNumericString
          .charAt(index));
    }

    return sb.toString();
  }


  public static void main(String[] args) throws InterruptedException {

    List<String> elements = new ArrayList<>();
    Random random = new Random();
    Runnable runnable = () -> {
      while (true) {
        elements.add(getAlphaNumericString(5));
        try {
          Thread.sleep(100);
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      }
    };
    Thread t = new Thread(runnable);
    t.start();

    logger.info("Thread started and filling..");

    Flux<String> source = Flux.fromIterable(elements)
        .filter(s -> s.length() == 5)
        .map(String::toUpperCase).subscribeOn(Schedulers.parallel());

    ConnectableFlux<String> connectable = source.publish();
    while (true) {
      connectable.subscribe(new Subscriber<String>() {
        private Subscription su;
        int onNextAmount;
        @Override
        public void onSubscribe(Subscription subscription) {
          this.su = subscription;
          su.request(2);
          logger.info("onSubscribe");

        }

        @Override
        public void onNext(String s) {
          logger.info("onNext");
          elements.remove(s);
          onNextAmount++;
          if (onNextAmount % 2 == 0) {
            su.request(2);
            logger.info("onNext:requested");
          }

        }

        @Override
        public void onError(Throwable throwable) {

        }

        @Override
        public void onComplete() {

        }
      });
      connectable.connect();
      Thread.sleep(1000);
    }

  }

}
```
