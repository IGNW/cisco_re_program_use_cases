# Bulk Deploying Controls of an IFTT Device via CE endpoint

### Requirements:

* *IFTTT Compatible Device* To make this guide, I used a WeMo Mini Wi-Fi Smart Plug that I got from Staples (Staples Item # 2722085 - MFR Item # F7C063) via in [smartbuy](https://s1.ariba.com/Buyer/Main?realm=cisco-child&guidedbuyredirect=true) (the replacement for iprocurement). Any IFTTT compatible device should work very similar. I've chosen to use IFTTT because many home automation devices don't provide an easy to use REST API, so IFTTT becomes a simile broker. Note in another guide I provide instructions for a TP-Link device using a more direct API.

* *CE based endpoint.* If you don't have admin rights on your codec and its registered to ACE, open a case with ace (you can use the "ACE Cisco Webex Device Bot"), and they will give it to you. You can demo CE Deploy against an endpoint in the DevNet sandbox, but today Internet outbound is disabled in the sandbox, so this flow with the smart plug will not work.

### Overview:

In the steps 4 & 5 you will upload a UI change and macro to a single device and in the step 5 you'll see a more automated method to deploy. But before we get there, we need to be able to control your smart plug from code.

## Step 1 - Setup IFTTT

1. If you do not already have an account with IFTTT then you can do so at https://ifttt.com/join

2. Assuming you have already set up your automation device, you will need to connect it to IFTTT. Devices may vary, some will be easily set up from with in IFTT by [searching](https://ifttt.com/search?tab=services) for the manufacture of the device (make sure you are looking in services not connections). In my case, WEMO requires you connect IFTTT from within their app. So I followed the "Connect to our Smarthome Partners" in the config section of their application.

3. To allow an inbound REST call to to be the trigger ("if this") portion of the applet you are going to  create, you are going to turn on the [webhooks](https://ifttt.com/maker_webhooks) function inside of IFTTT.  Navigate to the [webhooks](https://ifttt.com/maker_webhooks) page and select Connect.

4. Your webhook has a unique URL/Key. Clicking on the "Documentation" button in the upper right will open a new window. Keep this window open and/or jot your key down. It uniquely identifies your account. _Note that URL provided is missing a key element, a name for your event._

## Step 2- Create your Applet(s)

In this example I am going to create two applets: one applet to turn my WeMo outlet on and one to turn it off.  This will correlate to a toggle button with two positions on my codec. _Note: If your automation device is more complex (for example choosing multiple colors of a lightbulb), you may require more triggers or to pass some data in the request._

1. Click on the [create page](https://ifttt.com/create) where you will see you will need to define the trigger ("if this") portion and the final action ("then that") of your applet.

2. Click the + next to "This" and search for the webhooks service.

3. There is likely only one trigger in the webhook, "Receive a web request", so select it.

4. The event name is the one field you need to configure on the webhook. Where the previous key identifies your account, this field which applet should be triggered.  _Note the field IS case sensitive._  For my simple on/off example I chose the name of wemo_on for my first applet (later I will create one named wemo_off).

5. Once you've saved the name, click on the + next to "That" and search again for your device manufacture. Upon searching for WEMO I found several devices and had to click on the WeMo Smart Plug.

6. Devices will vary quite a bit on this next page depending on their capability. For a WeMo switch, "Turn On" is the obvious choice for the first applet. After selecting the action you may be asked which device (or group of devices) should be triggered.

7. Click on "Create Action" and you will be at the "Review and finish" page. Feel free to turn on or off the notifications then click "Finish."

8. Navigate back to the tab you kept open with your Key on it. It's time to test your applet. Click on the {event} name and type in the unique name of your event, in my case that was wemo_on. Now click "Test It" and the applet should trigger.

9. Create additional applets as needed by repeating 1 through 8. For my WeMo, this was just one more wemo_off applet.


## Step 3 - Test Command Line

1. SSH into your codec. _Note: You could also use “Developer API” screen to input these commands._ For ACE, usually the username is sales so an example from a OSX terminal would be:
```
ssh sales@<IP Address>
```
2. Typically the http client built into the code is turned off. You must enable the HttpClient using a xConfiguration command (or via the web interface):
```
xConfiguration HttpClient Mode: On
```
3. You can now run another xcommand to send the same http get that we did in the previous step from the IFTT page.

```
 xCommand HttpClient Get Url: "https://maker.ifttt.com/trigger/wemo_on/with/key/dakLfx2346731279v3UH" Header: "Content-Type: application/json"
 ```
_Note you need to replace the above URL with your unique URL including the unique event name.
Also Note that DevNet sandbox do not allow outbound http._

## Step 4 - Create a Button


1. We are going to create a very simple button that has a toggle for on or off. _Note, I am providing the XML for an ON/OFF example, if you have a more complex set of actions, you will need to create your own button. This is easiest to do on the Codec, but actually can be done via code._
Open a text editor and save the following as an XML file.
```
<Extensions>
  <Version>1.6</Version>
  <Panel>
    <PanelId>Light</PanelId>
    <Type>Statusbar</Type>
    <Icon>Lightbulb</Icon>
    <Order>1</Order>
    <Color>#FFB400</Color>
    <Name>Light</Name>
    <ActivityType>Custom</ActivityType>
    <Page>
      <Name>Light Control</Name>
      <Row>
        <Name>Light1</Name>
        <Widget>
          <WidgetId>light</WidgetId>
          <Type>ToggleButton</Type>
          <Options>size=1</Options>
        </Widget>
      </Row>
      <PageId>Light Control</PageId>
      <Options>hideRowNames=1</Options>
    </Page>
  </Panel>
</Extensions>
```
2. Open the UI editor from the Code's admin page under Integration -> UI Extension Editor
3. Use the 3 lines in the top right corner to Import from file the XML you saved in the previous step.
4. Feel free to adjust the labels or icons. My WeMo has a light plugged into it, so yo will see references to a light. _Note if you change the "Widget Id", you will also need to change the code in a later step._
5. Once done with any changes export the control to the video system with the up arrow pointing to a line. You should see the button on the device and an ok message from the system.
6. (Optional) Its probably best at this point to test that the button even though we haven't added the macro yet. SSH into the codec and type the following:
```
xFeedback Register Event/UserInterface/Extensions/Widget/Action
```
Now when you press the light button (or whatever you called it) and switch it on or off, the codec should report the change (even if nothing else happens). If you are not using the on/off example this would be a good way to see what output is given before writing your macro.

## Step 5 - Upload the Macro

1. Open a text editor and save the following with a .js (JavaScript) extension. You will need to change the key on the 5th line (where I have lodfkh9-248hnsdaf) and your event names on lines 13 & 16 (where I have wemo_on and wemo_off).

```
const xapi = require('xapi');

function wemo(event_name) {
  // change the state of the wemo smart plug
  var url = 'https://maker.ifttt.com/trigger/' + event_name + '/with/key/lodfkh9-248hnsdaf';
  var headers = 'Content-Type: application/json';
  xapi.command('HttpClient Get', { 'Url': url, 'Header': headers });
}

function light(event) {
// did the user press on or off?
  if (event.WidgetId === 'light' && event.Type === 'changed' && event.Value === 'on') {
    wemo("wemo_on");
  }
  if (event.WidgetId === 'light' && event.Type === 'changed' && event.Value === 'off') {
    wemo("wemo_off");
  }
}
// wait for a button to be depressed
xapi.event.on('UserInterface Extensions Widget Action', light);
```
This code reads somewhat from the bottom up. The last line waits for a user to input something (should look very similar to the xfeedback register command you did from SSH previously), the 2nd function "light" determines if the command is toggling on or off, and the first function, sends the appropriate command to the IFTTT cloud (again will look very similar to xCommand HttpClinet Post that you previously sent via SSH).

2. Go back to the main admin page for your codec and go to Integration -> Macro editor
3. Macro's may be disabled on the system by default, if so, you will be prompted to (and should) enable them.
4. Use the "Import from file" command to import the code you saved above.
5. Save (disk icon) the upload and then use the radio button to toggle the macro on.
6. You should now be able to toggle the button and turn the smart plug on. Note for demo purposes, you can also go back to the macro editor to show the button (and toggle it) from the web.

## Step 6 - Bulk Deployment Using CE Deploy

Further use the APIs by leveraging CE deploy to bulk configure multiple endpoints.

1. Take a look at the CE Deploy github page [here](https://github.com/voipnorm/CE-Deploy), but instead of using the install instructions included, its easiest just to download the install pre-compiled files from Chris' box folder. The link changes so [join his Webex teams space](https://eurl.io/#SJWfk6qUV) to get the latest. 
2. CE deploy needs a csv file containing all of the devices you wish to bulk configure. Assuming for demo purposes you just want to configure one endpoint, create a csv in spreadsheet tool like excel with the only item as the IP address of your codec (note by opening CE deploy and hitting the top 3 bar menu you will find instructions including more details around the csv file).
3. Navigate to the codec where you initially created your macro and ui control.
4. Open the macro editor once more. This time use the wrench icon next to the name of your macro and first export it to a file then delete it.
5. Do the same thing for the UI by first going to the UI extension editor, using the 3 line button to export your extension to a file, then deleting the UI.
4. Open CE deploy and select the csv file you created as the "endpoint file".
7. Enter valid credentials on your endpoint and test that your authentication is working.
8. Upload (and name) the UI/Panel you created (not config, see CE instructions on the difference)
9. Upload (and name) your macro. Also select activate on deployment.
10. Test that your macro works again. If you have trouble with CE deploy, there is a [Webex team space](https://eurl.io/#SJWfk6qUV) you can join for more assistance.
