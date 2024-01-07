# SolaX Charge Time
The files in this section cover setting up an automation in Home Assistant to automatically adjust the force charge time of the SolaX inverter to the lowest cost 2 hour window on an Octopus tariff.

**Caveats:** this works for me on Octopus Agile with a SolaX X1-Hybrid-G4 inverter and Home Assistant on a Pi talking to a Waveshare. YMMV!

The key steps are (assuming HA is set up with the SolaX and Octopus integrations working):
1. Create a target rate sensor to find a low-cost 2 hour window;
1. Trigger an automation when the next day's Octopus rates are published and a new window is identified;
1. Update the charge time on the inverter for the new window.

**Note:** Make sure you have the *File editor* Add-on installed in HA!

## TODO
* Probably lots - this is a first attempt at anything like this with HA and I'm new to SolaX as well...
    * The script paramaters need tightening up though to allow UI mode for testing. It should turn "HH:MM:SS" into "HH:MM" for you...

## Setup

### Configure the Inverter
For me this all works with the inverter running in "self-use" mode with grid charging enabled.

- In the SolaX app select the inverter;
- On the inverter screen select the *"settings"* option (the cog in the top right corner);
- Select *"Work Mode"* and make sure it's set to *"Self Use"*.
    - If not, update it and select save;
- Select *"Setting"* to enter the User Settings section;
- In the *"Self Use"* section set:
    - *"Min Soc (%)"* to your minimum battery charge level you want to allow (mine is at 10%);
    - *"Charge battery to (%)"* to your maximum battery charge level (mine is 100%);
    - *"Charge from grid"* to *"Enable"* to allow the battery to be charged from the grid as well as solar.
- In the *"Chrg&Dischrg Period"* section:
    - I have the *"Allowed Disc Period Start Time"* set to *"00:00"* and the *"Allowed Disc Period End Time"* set to *"23:59"* - essentially allow the battery to be used any time there's insufficient solar power;
    - The *"Forced Charge Period Start Time"* and *"Forced Charge Period End Time"* are the values this automation will update - you can adjust these manually if you want to make sure everything works!
    - *"Chrg&Dischrg Period2"* I have disabled - if you enable this you may need to make sure the two charge periods don't clash!


### Configure SolaX Modbus
- Install the [Solax Inverter Modbus](https://github.com/wills106/homeassistant-solax-modbus) integration into Home Assistant;
    - Get this up and running!
    - If you want to test at this stage go to the *Developer Tools* in HA and the *STATES* tab and type "charger" in the *"Filter entities"* box at the top of the *Entity* column;
        - You should get several entities back including *select.solax_charger_start_time_1* and *select.solax_charger_end_time_1* which should (might!) have the values you saw in the SolaX app earlier;
        - You can click on these and change the state value and see it reflected in the app - but note that the value must be one of the **list of options** through the HA integration. They can't be set to (or display) arbitrary times. (See [screenshot](./screenshots/solax_charger_start_time_1%20entity%20with%20options.png))
    - If this doesn't show the updates in the app then you might need to unlock the inverter first or it might be in *"Idle"* mode in which case you might need to wait for some PV!


### Create Octopus Target Rate Sensor
- Install the [Octopus Energy](https://github.com/BottlecapDave/HomeAssistant-OctopusEnergy) integration into Home Assistant;
    - Get this up and running!
- In Home Assistant (HA) go into *Settings|Devices & Services*, find *"Octopus Energy"* in the integrations and select it;
    - You should see a section called *"Integration entries"* with an *"ADD ENTRY"* option at the bottom;
- Click on *"ADD ENTRY"* to create a target rate sensor:
    - I used *"charge_battery"* in the first field (name),
    - *"2.0"* in the *"hours you require"* field - for a 2 hour window,
    - *Continuous* - to find a single 2 hour block,
    - Your MPAN,
    - *"00:00"* for the minimum time - to search from midnight,
    - *"07:00"* for the maximum time - to search until 7am
    - No offset and nothing else checked (tweak the above values to be appropriate for you!)

You should now be able to find a new entity called *binary_sensor.octopus_energy_target_charge_battery* with a load of attributes.


### Create a Script to Update Charge Time 1
I chose to create a script to handle making the actual changes to the charge time on the inverter. That way I can call it manually (e.g. for testing) or from the automation.

- In HA go to *Settings|Automations & scenes* and select *"Scripts"*;
- Select *"+ ADD SCRIPT"* followed by *"Create new script"*;
- In the script editor that appears select the three dot menu in the top right corner and select *"Edit in YAML"*;
- Copy and paste the [Update Charge Time 1](./scripts/updateSolaxChargeTime1.yaml) script into the editor (replacing all the existing content) and save it.
- Go back to the scripts page and you should now have an *"Update Solax Charge Time 1"* script listed.
    - Select the three dot menu on the right for this script and pick *"Information"*;
    - In the information dialog select *settings* (the cog icon);
    - On the settings page make a note of the numeric value part of the *"Entity ID"* after *"script."* - we want to change this to something more useful.

- In HA open the *File Editor* add-on and select *browse file system* (the folder icon at the top).
    - Navigate to the *config/* part of the tree and open *scripts.yaml*
    - Scroll through the file and find the entry for the script we've just created - it will start with the numeric part of the entity ID from above, e.g. `'1699395551527':`
    - Change this to a more friendly `update_solax_charge_time_1:` and save the file

- In HA go to the *Developer tools* and *YAML* tab;
    - Scroll down to the *SCRIPTS* entry and select it to reload the file we've just edited

You should now be able to find this script through the *SERVICES* tab of the *Developer tools* and test it by suppliying sample values. **Note:** if you use the *UI mode* it will give you a time picker for the start_time and end_time parameters, *but* they only work with *"HH:MM"*. You may need to switch to *YAML mode* and change it to be something like:

```
service: script.update_solax_charge_time_1
data:
  start_time: "04:00"
  end_time: "06:00"
```

There are some sample [screenshots](./screenshots/) (*"Call Script..."*).

If the script is working then when you call it you should get a bunch of notifications to let you know what it's done (you might want to remove some to make it less chatty!) and you should be able to confirm the changes through the SolaX app.

### Create an Automation to Call the Script
Now we have a script to update the inverter settings the last step is to set up an automation to call it when the next day's rates are available. We can do that easily by triggering on a change to the *next_time* attribute of the target sensor we created earlier (*binary_sensor.octopus_energy_target_charge_battery*).

Before doing that create a helper to record the average cost (I display this on my HA dashboard and it gets lost once the target time passes)
- In HA go to *Settings|Devices & services* and select *"Helpers"*;
- Select *"+ CREATE HELPER"* and choose *"Number"*;
    - Call it "charge_battery_average_cost" and make sure it has a suitable positive *and* negative range (I've used -100 -> +100). I've also used a step size of 0.1 as tenth of a penny is more than accurate enough for my needs! (See [Average Cost Helper](./screenshots/Average%20Cost%20Helper.png) )

Now create the automation:
- In HA go to *Settings|Automations & scenes* and select *"Automations"*;
- Select *"+ CREATE AUTOMATION"* followed by *"Create new automation"*;
- In the script editor that appears select the three dot menu in the top right corner and select *"Edit in YAML"*;
- Copy and paste the [Update Grid Charge Period](./automations/update_grid_charge_period.yaml) automation into the editor (replacing all the existing content) and save it.
    - Call it whatever name you like, I used "Update Grid Charge Period"
- Go back to the automations page and you should now have an *"Update Grid Charge Period"* automation listed.

You should now be all set and the next time the rates change the automation should fire and you should get a bunch of notifications.

**If** your target sensor already has the window identified (i.e. if you look at the entity in the *Developer tools* and it has a timestamp for the *next_time* attribute) then you can manually run the automation through the three dot menu on the right. That should then simulate being triggered and should all work!