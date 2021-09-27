# Laravel Zoom

## Laravel Zoom API Client

![Header Image](https://github.com/MacsiDigital/repo-design/raw/master/laravel-zoom/header.png)

<p align="center">
 <a href="https://github.com/MacsiDigital/laravel-zoom/actions?query=workflow%3ATests"><img src="https://github.com/MacsiDigital/laravel-zoom/workflows/Tests/badge.svg" style="max-width:100%;"  alt="tests badge"></a>
 <a href="https://packagist.org/packages/macsidigital/laravel-zoom"><img src="https://img.shields.io/packagist/v/macsidigital/laravel-zoom.svg?style=flat-square" alt="version badge"/></a>
 <a href="https://packagist.org/packages/macsidigital/laravel-zoom"><img src="https://img.shields.io/packagist/dt/macsidigital/laravel-zoom.svg?style=flat-square" alt="downloads badge"/></a>
</p>

Laravel Zoom API Package

## Installation

You can install the package via composer:

```bash
composer require macsidigital/laravel-zoom
```

For versioning:-

- 1.0 - deprecated - was a quick build for a client project, not recommended you use this version.

- 2.0 - Laravel 5.5 - 5.8 - deprecated, no longer maintained

- 3.0 - Laravel 6.0 - Maintained, feel free to create pull requests.  This is open source which is a 2 way street.

- 4.0 - Laravel 7.0 - 8.0 - Maintained, feel free to create pull requests.  This is open source which is a 2 way street.

### Configuration file

Publish the configuration file

```bash
php artisan vendor:publish --provider="MacsiDigital\Zoom\Providers\ZoomServiceProvider"
```

This will create a zoom.php config file within your config directory:-

```php
return [
    'apiKey' => env('ZOOM_CLIENT_KEY'),
    'apiSecret' => env('ZOOM_CLIENT_SECRET'),
    'baseUrl' => 'https://api.zoom.us/v2/',
    'token_life' => 60 * 60 * 24 * 7, // In seconds, default 1 week
    'authentication_method' => 'jwt', // Only jwt compatible at present
    'max_api_calls_per_request' => '5' // how many times can we hit the api to return results for an all() request
];
```

You need to add ZOOM_CLIENT_KEY and ZOOM_CLIENT_SECRET into your .env file.

Also note the tokenLife, there were numerous users of the old API who said the token expired to quickly, so we have set for a longer lifeTime by default and more importantly made it customisable.

That should be it.

## Usage

Everything has been set up to be similar to Laravel syntax. So hopefully using it will be similar to Eloquent, right down to relationships.

Unfortunately the Zoom API is not very uniform and is a bit all over the place.  But we have hopefully made this uniform and logical. However you will still need to read the [Zoom documentation](https://marketplace.zoom.us/docs/api-reference/introduction) to know what is and isn't possible.

At present we cover the following modules

- Users
- Roles
- Meetings
- Past Meetings
- Webinars
- Past Webinars
- Recordings

Doesn't look like a lot but Meetings and Webinars are the 2 big modules and includes, polls, registration questions, registrants, panelists and various other relationships.

Also note that some of the functionality is only available to certain plan types.  Check the [Zoom documentation](https://marketplace.zoom.us/docs/api-reference/introduction).

### Connecting

use below function according to your desire it will create start url of zoom meeting and joiing url of zoom meeting

``` php
   <?php

namespace App\Http\Controllers;

use App\Jobs\SendZoomMeetingLinkJob;
use App\Models\Event;
use App\Models\EventRequest;
use App\Models\User;
use App\Models\ZoomMeeting;
use Illuminate\Http\Request;
use MacsiDigital\Zoom\Facades\Zoom;

class ZoomIntegrationController extends Controller
{
    public function meeting_create(Request $request)
    {


        $events=EventRequest::with('user')->where('event_timing_id',$request->event_time_id)->where('status','active')->get();
        $event_generater=Event::find($request->event_id);
        if(Zoom::user())
        {

            $user = Zoom::user()->first();

            $meeting = Zoom::meeting()->make([
                'userId' => 'me',
                'topic' => 'Online Class',
                'type' => 2,
                // date('Ý-m-dTH:i:sZ', strtotime($request->start_time))
                // 'start_time' => '2020-03-31T12:02:00Z',
                'disable_recording' => false,
                'duration' => 1,
                'timezone' => 'Pakistan',
                'password' => '1234567890',
                'agenda' => 'Meeting for Event Discussion',
            ]);

            if(!$meeting)
            {
                return redirect()->back()->with('message','There is no meeting, only zoom user can create meeting');
            }
            else
            {
                $test= $user->meetings()->save($meeting);

                ZoomMeeting::create([
                   'event_timing_id'    =>$request->event_time_id,
                   'event_id'           =>$request->event_id,
                   'joining_url'        =>$test->join_url,
                   'start_url'          =>$test->start_url,
                ]);

                foreach ($events as $event)
                {
//                    sendMail([
//                        'view' => 'email.advocate.zoom_meeting',
//                        'to' => $event->user->email,
//                        'name' => 'Zoom Meeting ',
//                        'subject' => 'Zoom Meeting Link',
//                        'data' => [
//                            'join_url' => $test->join_url,
//                            'start_url'=>$test->start_url,
//                            'event_generater'=>$event_generater,
//                            'event'=>$event,
//                        ]
//                    ]);
                    $data=array([
                        'join_url' => $test->join_url,
                        'start_url'=>$test->start_url,
                        'event_generater'=>$event_generater,
                        'event'=>$event,
                    ]);

                    dispatch(new SendZoomMeetingLinkJob($data))->delay(now()->addSeconds(2));
                }

            }
        }
        else
        {
            return redirect()->back()->with('message','There is no zoom user, please try again');
        }

        return redirect()->back()->with('message','The Meeting is created');
    }
}
```







