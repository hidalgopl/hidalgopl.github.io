# How to write DRF code if you believe in your startup success


In a fast-paced, digital world we all live in, time to market is often defined as one of the key factors for success. If you are creating a startup, you probably heard about Minimal Viable Product, most likely some smart people told you also to avoid premature optimization. Just focus on delivering value, right? Don’t solve problems you don’t have yet. Pick a tool, language or framework which we’ll help you to be productive quickly. Don’t reinvent the wheel. Build from open-source, ready, building blocks.

I have to say, popular backend frameworks, such as Django or Rails are often great tool to build sophisticated systems pretty quickly.

I heard it many times. While I agree with those, I’ve seen many teams struggling with maintaining and developing huge monolithic codebase. This usually happens if your startup achieved success, attract attention and get amount of web traffic which your hackish, DDD(deadline driven development) app can’t handle anymore.
Ooops. We’ve got a problem now. We took a lot of shortcuts and now is time to pay a tech debt.

As a co-founder or CTO, you need to take immediate action to solve this problem. You can hire more developers and start splitting your code into microservices, which would very likely be a huge effort and would stop developing new features for a while. If you weren’t very disciplined at the beginning, probably your monotlith app isn’t really ready for split. Looks like there are two ways which you can take now: You can try split it as it is, which would be a maintenance hell, or just trying to rewrite some parts from scratch, which can cause series of bugs if you didn’t invest in some end-to-end or integration tests at the beginning.

If you are in similar situation right now and your weapon of choice was Django, I can imagine you may have issues like:
1. Huge models with a lot of business logic inside (VERY BAD, although a lot of books were promoting “Fat models” as best practice not so long ago)
2. Huge serializers with business logic inside.
3. Gigantic viewsets, also with business logic inside
4. Business logic separated, but relaying on django features (such as models, serializers, signals, etc.)
5. No unit tests at all
6. No tests at all
7. Integration tests only, which require spinning test database, test server and going through all layers of app (view, serialization, model) and mocking like crazy, just to test business logic.

If any of those sounds familiar, keep reading.

Those are common sins I encountered in django apps during my career. All those are significant blockers on your way to microservices architecture.

But hey! We can do better than this since very beginning, without investing much extra time. In this post, I’ ll show you few basic techniques how to do this. We will use well known Object oriented programming techniques, which is Hexagonal architecture, a.k.a ports-adapters architecture. 

Let’s focus on real world example (little bit simplified for sake of readability).

Imagine you are developing assets portfolio app. User can have different types of assets, such as stock shares, gold/silver coins or just foreign currencies. Your app role is to tell him how much are those worth now.
Let’s say user can set his assets manually and we will persist amounts of different types of assets in database. You want to add an endpoint, which would return current value of his assets. To make him happy, we also need to show how much he gained from investment. To get up-to-date currencies rates, we will need to call 3rd party API. We’ll use this API to get rates from the day when he invested a money, just to compare those and show a difference (hopefully, they are worth now more than 100%).

To make this examples more readable, let’s make some assumptions:
1. User want to see final calculated value in US Dollars.
2. User invested in CHF and EUR.

To sum up, here’s expected response:
```
GET porfolio/my

{ “invested_value_in_usd”: 2356.23, “current_value_in_usd”: 2444.23, “current_value_percentage”: 101.23}
```

This is how our Portfolio model can look like:

`myapp.models.py`
```python
from django.db import models

class Portfolio(models.Model):
    user = models.ForeignKey("auth.User")
    chf_amount = models.FloatField()
    eur_amount = models.FloatField()
    chf_rate = models.FloatField()
    eur_rate = models.FloatField()
```


Now, you encountered first issue. When Portfolio model is created, we want to query 3rd part API for current rates and save them to DB, just to avoid making those calls each time. You’re probably tempted to override save functions in model, right? 
We can add also a function, which would calculate current portfolio value in $$.
Let’s see how it would look like:

`myapp.models.py`
```python
from django.db import models
import requests


class Portfolio(models.Model):
    user = models.ForeignKey("auth.User")
    chf_amount = models.FloatField()
    eur_amount = models.FloatField()
    chf_rate = models.FloatField()
    eur_rate = models.FloatField()
    
    def save(self, *args, **kwargs):
        self.super()
        if self.pk is None:  # this will be True only if portfolio is getting created
            resp = requests.get("https://currenciesapi.com/rates?=CHF,EUR&base=USD")  # this is fake url
            self.chf_rate = resp.json()["CHF"]
            self.eur_rate = resp.json()["EUR"]
        return self
   
    def calculate_starting_value(self):
        return self.eur_rate * self.eur_amount + self.chf_rate * self.chf_amount

    def calculate_current_value(self):
        # if you’re a bit smarter, you probably use cached_property decorator on this function
        # I’m not doing it here, because whole idea about keeping it in model is WRONG.
        resp = requests.get("https://currenciesapi.com/rates?=CHF,EUR&base=USD")  # this is fake url
        body = resp.json()
        current_eur_rate = body["EUR"]
        current_chf_rate = body["CHF"]
        return current_eur_rate * self.eur_amount + current_chf_rate * self.chf_amount
```



You may think, OK, that’s good enough, now my serializers and views would be way cleaner.

Let’s see how they would look like:

`myapp.serializers.py`
```python
from rest_framework import serializers

from myapp.models import Portfolio

class PortfolioSerializer(serializers.ModelSerializer):
    invested_value_in_usd = serializers.SerializerMethodField()
    current_value_percentage = serializers.SerializerMethodField()
    current_value_in_usd = serializers.SerializerMethodField()
    
    def get_invested_value_in_usd(self, obj):
        return obj. calculate_starting_value()
    
    def get_current_value_in_usd(self, obj):
        return obj.get_current_value()
    
    def get_current_value_percentage(self, obj):
        return obj.get_current_value() * 100 / obj.calculate_starting_value()
       
     
    class Meta:
        model = Portfolio
        fields = ("invested_value_in_usd", "current_value_percentage", "current_value_in_usd")
```
`myapp.views.py`
```python
from rest_framework.mixins import RetrieveModelMixin

from myapp.models import Portfolio
from myapp.serializers import PortfolioSerializer


class PorfolioView(RetrieveModelMixin):
    serializer_class = PortfolioSerializer
    queryset = Portfolio.objects.all()
```


So this covers almost all bad practices I mentioned earlier. It’s even worse: there’s bunch of duplicated code (e.g. `resp = requests.get(“”)), hardcoded urls. It is also not optimal in terms of performance, as it calculates starting value each time. This code also make it impossible to write unit tests. To test this feature, you basically have to write integration test, that will need created Portfolio object in database at the beginning, APITestClient to call endpoint and also, some kind of patching / mocking to avoid querying 3rd party API during tests. Let’s see how we can do better.

What would you say about taking two steps back and reconsidering what’s really our business logic in this case?

In my eyes, it’s simple math operations:

```python
chf_amount * chf_rate + eur_amount * eur_rate
```
Once we have that, we also need to compare those with invested_value to calculate percentage.
Why don’t we just write it from scratch here? We’ll use python dataclasses, to avoid relaying on django model.

```python
from dataclasses import dataclass

@dataclass
class PortfolioDTO:
    eur_amount: float
    chf_amount: float


@dataclass
class RatesDTO:
    eur_rate: float
    chf_rate: float
```

Wait, but this may feel weird. We had almost the same data structure in django model, right? Why then we write it over again? Bear with me, and I hope you’ll have “aha” moment soon.

Those two dataclasses serve two purposes: PortofolioDTO will let us abstract data source of user's assets (as it may change) and RatesDTO abstracts source of currency exchange rates.
That's useful, because we may need to switch to different underlying database, ORM or framework as well as we can use different API to fetch currency rates.
And I can tell you with 100% certainty that we want to use different data sources in our tests.

Now, let's wrap our currency API client into helper class, which will implement pretty clear interface:

`myapp.clients.py`
```python
from abc import ABC, abstractmethod
from dtos import RatesDTO
import requests

class HttpClientAdapter(ABC):
    @abstractmethod
    def get(self, url: str) -> dict:
         pass

class RatesClient(ABC):
    @abstractmethod
    def get_current_rates(self) -> RatesDTO:
        pass

class CurrencyRatesClient:
    def __init__(self, http_client=requests.Client):
        self.currencies = "EUR,CHF"
        self.base_currency = "USD"
        self.client = http_client()
        self.base_url = ""
   
   def build_url(self):
        return f"{self.base_url}?={self.currencies}&base={self.base_currency}"

    def get_current_rates(self) -> RatesDTO:
        url = self._build_url()
        resp = self.client.get(url)  # normally you want to add try/except error handling here – I’m not doing it for sake of simplicity of this example
        body = resp.json()
        return RatesDTO(
            eur_rate=body["EUR"],
            chf_rate=body["CHF"]
        )
```
Goal here is to avoid writing same code lines over and over, as well as make it easier to mock 3rd Party API requests in tests. In any time, I also want to have ability to change http client (now I’m using requests, but in the future – who knows?)
So, we have nice clean Client class to get current rates. We can easily unit test it without need to send any http requests, because we pass http_client class as an argument in initialization. We can easily pass mocked class there, all it has to have is mocked `get(url: str) → dict` method.
At some point you may want to change framework and use async http client, but guess what?
You can reuse CurrencyRatesClient class. Just create AsyncHttpClientAdapter with get method and pass it CurrencyRatesClient. Pretty neat, ha?

But there’s more!

Now we need adapter that will translate django model instance into our `PortfolioDTO`
It's really simple one.
`myapp.adapters.py`
```python
from my_app.models import Portfolio
from my_app.dtos import PortfolioDTO


def portfolio_django_adapter(portfolio_obj: Portfolio) -> PortfolioDTO:
    return PortfolioDTO(eur_amount=portfolio_obj.eur_amount, chf_amount=portfolio_obj.chf_amount)
```

Finally, we’re getting to business logic handler class, which will connect all of the puzzles.

`myapp.dtos.py`
```python
from dtos import PortfolioDTO, RatesDTO

class PortfolioHandler:
    def __init__(self, rates: RatesDTO, portfolio: PortfolioDTO):
        self.rates = rates
        self.portfolio = portfolio

    def get_current_value(self):
        return self.rates.eur_rate * self.portfolio.eur_amount + self.rates.chf_rate * self.portfolio.chf_amount
```

Now, this is how the view looks like:

`myapp.views.py`
```python
from rest_framework.generics import GenericAPIView
from rest_framework.response import Response

from myapp.adapters import portfolio_django_adapter
from myapp.clients import CurrencyRatesClient
from myapp.handlers import PortfolioHandler
from myapp.models import Portfolio


class PorfolioView(GenericAPIView):
    def get(self, request, *args, **kwargs):
        portfolio = self.get_obj_or_404(Portfolio, user=request.user)
        starting_value = portfolio.calculate_starting_value()
        portfolio_dto = portfolio_django_adapter(portfolio)
        rates_client = CurrencyRatesClient()
        rates_dto = rates_client.get_current_rates()
        handler = PortfolioHandler(rates=rates_dto, portfolio=portfolio_dto)
        current_value = handler.get_current_value()
        return Response({"current_value_in_usd": current_value, "invested_value_in_usd": starting_value, "current_value_percentage": (current_value * 100) / starting_value})

```

Hmm, it looks pretty nice, but it was a lot of effort. And what for?
Well, short answer is, for this:

`tests.py`
```python
import pytest
from myapp.dtos import PortfolioDTO, RatesDTO
from myapp.handler import PortfolioHandler

@pytest.mark.parametrize(
    "portfolio_dto,rates_dto,expected_value",
    [
        (PortofolioDTO(eur_amount=1000.00, chf_amount=500.00), RatestDTO(eur_rate=1.0, chf_rate=2.0), 2000.00]
)
def test_portfolio_handler(portfolio_dto, rates_dto, expected_value):
    handler = PortfolioHandler(rates=rates_dto, portfolio=portfolio_dto)
    assert expected_value == handler.get_current_value()
```

Long answer: to have ability to write simple, comprehensive unit tests. To have ability to just copy this code and run it in different framework, with different ORM, without relying on those. How simple is that? Well, all you need to do is create another `adapter` function, which will translate this another ORM object to our Data Transfer Object.
Now we have portable, simple, well tested module with our business logic. Reading one unit test is now all it takes to understand what's going on on this endpoint.


To sum up:
This approach may seem to you as a lot of boilerplate writing. Well, it's kinda true. So I recommend to use this approach in parts of your app that are essential and unlikely to change (from business perspective).
It may seem like boilerplate, but I believe it saves a lot of time _later_.
