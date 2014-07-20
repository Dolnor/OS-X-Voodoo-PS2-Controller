## Fork of RehabMan’s VoodooPS2Controller by TimeWalker

[Original project description and source] (https://github.com/RehabMan/OS-X-Voodoo-PS2-Controller)

### Changes:

- Letting ACPI namespace know what HID F-key mode is being used
- Add keyboard profile for Asus Eee PC 1201N
- Add Synaptics touchpad profile for DELL SNB-CPT and ASUS 1201N


The intention of this fork is to provide isolated behavior for EC queries when using Special and Standard key modes. 
By implementing an ACPI method called MDTG (Mode Toggled) for PS2K device we can let ACPI namespace know that we have switched HID F-key mode.

		    // Variable used by VoodooPS2 to store kHIDFKeyMode
                    Name (MODE, One)
                    Method (MDTG, 1, NotSerialized)
                    {
			// Zero is stored for Special Mode
			// One is stored for Standard Mode
                        Store (Arg0, MODE)
                    }

Knowing the value of MODE variable EC queries can therefore execute dedicated scenarios/produce ps2 scancodes based on active key mode used. 

At boot, driver passes 1 to MDTG when Standard mode is used and 0 when Special mode is used. The notification to ACPI namespace also happens when user changes the setting in Keyboard preference pane.

An example of such use can be an Fn-combo that wouldn’t normally generate a scancode, because it relies on WMI notifications. In Special mode we can assign RKA0-RKAF ACPI method to be evaluated by Voodoo driver thus causing certain event to happen:

		    // Method called in special key mode when F2 is pressed
	            // to indicate that Wi-Fi and BT need to be toggled
                    Method (RKA0, 1, NotSerialized)
                    {
                        If (Arg0)
                        {
                            RFKL () // Call RF Kill Toggle
                        }
                    }

		    // Method called in special key mode when F7 is pressed
	            // to indicate that LCD backlight needs to be toggled
                    Method (RKA1, 1, NotSerialized)
                    {
                        If (Arg0)
                        {
                            LBTG () // Call LCD Backlight Toggle 
                        }
                    }

In Standard mode the event is triggered by respective EC query. With changes proposed in this fork, one could rewrite the EC query with isolated actions for both key modes without the need of duplicating or renaming the query:

			// Fn+F2 issues RF kill/toggle event
                        Method (_Q06, 0, NotSerialized)
                        {
			    // In Standard Mode it needs to toggle BT and WF state
                            If (LEqual (^^PS2K.MODE, One))
                            {
                                ^^PS2K.RFKL ()
                            }
			    // We need it to generate a scancode in Special Mode for F2 to work
                            Else
                            {
                                Notify (PS2K, 0x0207)
                                Notify (PS2K, 0x0287)
                            }
                        }

			// Fn+F7 issues LCD backlight toggle event
                        Method (_Q10, 0, NotSerialized)
                        {
			    // In Standard Mode it needs to toggle LCD backlight
                            If (LEqual (^^PS2K.MODE, One))
                            {
                                ^^PS2K.LBTG ()
                            }
			    // We need it to generate a scancode in Special Mode for F7 to work
                            Else
                            {
                                Notify (PS2K, 0x0208)
                                Notify (PS2K, 0x0288)
                            }
                        }


Another case could be a situation when there’s a dedicated button for certain action. For example a button that disables touchpad alongside an existing Fn-combo. In situation like that mapping e037 to some F-key will result in dedicated button also becoming the respective F-key when Special mode is used. Solution is plain simple:

		    // Method called in special key mode when F9 is pressed
	            // to indicate that touchpad needs to be toggled
                    Method (RKA2, 1, NotSerialized)
                    {
                        If (Arg0)
                        {
                            Notify (PS2K, 0x0237)
                            Notify (PS2K, 0x02B7)
                        }
                    }

Rewriting Fn+Fx EC query with isolated behavior for Special mode we can generate a scancode that is unique to this Fn-combo:

			// Fn+F9 issues touchpad toggle event
                        Method (_Q14, 0, NotSerialized)
                        {
			    // In Standard Mode it needs to notify VoodooPS2 that we need to disable touchpad
                            If (LEqual (^^PS2K.MODE, One))
                            {
                                Notify (PS2K, 0x0237)
                                Notify (PS2K, 0x02B7)
                            }
			    // Because there is a dedicated touchpad toggle button 
		            // we can’t just assign e037 to F9 or that button would turn into F9 as well
			    // so we need it to generate a dedicated scancode in Special Mode for F9 to work
                            Else
                            {
                                Notify (PS2K, 0x0209)
                                Notify (PS2K, 0x0289)
                            }
                        }

While dedicated button will retain it’s original functionality regardless of HID F0key mode:

			// Dedicated Touchpad toggle button issues touchpad toggle event
                        Method (_Q27, 0, NotSerialized)
                        {
			    // Notify VoodooPS2 (regardless of key mode) that we need to disable touchpad
                            Notify (PS2K, 0x0237)
                            Notify (PS2K, 0x02B7)
                        }


### How to Install:

Please read and follow the important instructions for installing in the wiki:

https://github.com/RehabMan/OS-X-Voodoo-PS2-Controller/wiki/How-to-Install


### Feedback:

Please use the following threads on tonymacx86.com for feedback, questions, and help:

http://www.tonymacx86.com/hp-probook/75649-new-voodoops2controller-keyboard-trackpad.html#post468941
http://www.tonymacx86.com/mountain-lion-laptop-support/87182-new-voodoops2controller-keyboard-trackpad-clickpad-support.html#post538365

Don’t post questions regarding this particular fork, if you have any problems feel free to open an issue.


### Credits:

All the awesome changes during these past years, specially PS2K device notification and kinetic scrolling for Synaptics: RehabMan
