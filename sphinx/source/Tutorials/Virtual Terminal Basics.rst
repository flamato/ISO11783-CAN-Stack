.. _VirtualTerminalBasics:

Virtual Terminal
===================

.. toctree::
   :hidden:
   :glob:

.. contents:: Contents
   :depth: 2
   :local:

This tutorial covers the basics on how to communicate with a Virtual Terminal (VT) Server using the CAN stack.

It is assumed you have completed all the other tutorials, as this tutorial covers application level use of the CAN stack.

The Virtual Terminal Client
----------------------------

The first step in communicating with a VT is creating an object called :code:`VirtualTerminalClient`. 
This object will act as your interface for all VT communication. 
The client requires two things to instantiate, a :code:`PartneredControlFunction` and a :code:`InternalControlFunction`. 
This is so that it can send messages on your behalf needed to maintain the connection and simplify the API over sending raw CAN messages to the VT.

Let's start our program similarly to the other tutorials, complete with the requisite control functions.

.. code-block:: c++

	#include "can_general_parameter_group_numbers.hpp"
	#include "can_hardware_interface.hpp"
	#include "can_network_manager.hpp"
	#include "can_partnered_control_function.hpp"
	#include "isobus_virtual_terminal_client.hpp"
	#include "socket_can_interface.hpp"

	#include <csignal>
	#include <iostream>
	#include <memory>

	static std::shared_ptr<isobus::InternalControlFunction> TestInternalECU = nullptr;
	static std::shared_ptr<isobus::PartneredControlFunction> TestPartnerVT = nullptr;
	std::vector<isobus::NAMEFilter> vtNameFilters;
	const isobus::NAMEFilter testFilter(isobus::NAME::NAMEParameters::FunctionCode, static_cast<std::uint8_t>(isobus::NAME::Function::VirtualTerminal));
	static SocketCANInterface canDriver("can0");

	using namespace std;

	void signal_handler(int signum)
	{
		CANHardwareInterface::stop();
		exit(signum);
	}

	void update_CAN_network()
	{
		isobus::CANNetworkManager::CANNetwork.update();
	}

	void raw_can_glue(isobus::HardwareInterfaceCANFrame &rawFrame, void *parentPointer)
	{
		isobus::CANNetworkManager::CANNetwork.can_lib_process_rx_message(rawFrame, parentPointer);
	}

	void setup()
	{
		CANHardwareInterface::set_number_of_can_channels(1);
		CANHardwareInterface::assign_can_channel_frame_handler(0, &canDriver);

		if ((!CANHardwareInterface::start()) || (!canDriver.get_is_valid()))
		{
			std::cout << "Failed to connect to the socket. The interface might be down." << std::endl;
		}

		CANHardwareInterface::add_can_lib_update_callback(update_CAN_network, nullptr);
		CANHardwareInterface::add_raw_can_message_rx_callback(raw_can_glue, nullptr);

		std::this_thread::sleep_for(std::chrono::milliseconds(250));

		isobus::NAME TestDeviceNAME(0);

		// Make sure you change these for your device!!!!
		// This is an example device that is using a manufacturer code that is currently unused at time of writing
		TestDeviceNAME.set_arbitrary_address_capable(true);
		TestDeviceNAME.set_industry_group(1);
		TestDeviceNAME.set_device_class(0);
		TestDeviceNAME.set_function_code(static_cast<std::uint8_t>(isobus::NAME::Function::SteeringControl));
		TestDeviceNAME.set_identity_number(2);
		TestDeviceNAME.set_ecu_instance(0);
		TestDeviceNAME.set_function_instance(0);
		TestDeviceNAME.set_device_class_instance(0);
		TestDeviceNAME.set_manufacturer_code(64);
		vtNameFilters.push_back(testFilter);

		TestInternalECU = std::make_shared<isobus::InternalControlFunction>(TestDeviceNAME, 0x1C, 0);
		TestPartnerVT = std::make_shared<isobus ::PartneredControlFunction>(0, vtNameFilters);
		TestVirtualTerminalClient->initialize(true);
		std::signal(SIGINT, signal_handler);
	}

	int main()
	{
		setup();

		while (true)
		{
			// CAN stack runs in other threads. Do nothing forever.
			std::this_thread::sleep_for(std::chrono::milliseconds(1000));
		}

		CANHardwareInterface::stop();
		return 0;
	}

It's the same boilerplate we've done before, but note the following key things:

* We're using shared pointers (`std::shared_ptr`) to make our control functions. The VT client class requires shared pointers to your control functions, so we want to make sure we start with those.

* Our partner has been defined to be any device on the bus with a function code that identifies it as a VT (isobus::NAME::NAMEParameters::FunctionCode)

With those notes in mind, let's create our VT client:

.. code-block:: c++

	#include "isobus_virtual_terminal_client.hpp"

	static std::shared_ptr<isobus::VirtualTerminalClient> TestVirtualTerminalClient = nullptr;

	TestVirtualTerminalClient = std::make_shared<isobus::VirtualTerminalClient>(TestPartnerVT, TestInternalECU);

Now, we've got our client created, and we need to configure it. More specifically, we need to give it at least one object pool to manage.

Object Pools
-------------

An ISOBUS object pool is a serialized blob of objects defined in ISO 11783-6. Basically it's the data the comprises the GUIs you see on a VT when you connect an ISOBUS ECU. 
In other words, it's a way for a "headless" device to explain to a VT how to draw it's UI in a very consistent way that will be portable across machines and platforms.

This data is essentially a long list of objects that represent things to a VT, such as:

* Font attributes
* Bitmaps
* Strings
* Numbers
* Pointers
* Lines
* Rectangles
* Polygons
* Ellipses
* A few more complex objects, like bar graphs and graphics contexts

An ECU uploads this blob of objects to the VT along with their semi-hierarchical relationships, the VT checks to make sure the format is valid, then starts displaying it on its screen.
Then, the ECU can use a set of CAN messages to update what's shown on the screen, and likewise, the VT can use CAN messages to tell the ECU when things like button presses happen.

This back-and-forth communication is the foundation for ISOBUS implement applications, and the CAN stack aims to make this communication as easy as possible.

The `utility` folder in the CAN stack contains a helper function to read standard ISOBUS .iop files. These files are the industry standard way of storing the binary blob that is the object pool.

You can make one for yourself if you have access to an ISOBUS object pool designer tool of some kind, but for our purposes, one has been included in the examples folder for you called `VT3TestPool.iop`.

Let's add some code to our example to read in this IOP file, and give it to our VT client as our only object pool.

.. code-block:: c++

	#include "iop_file_interface.hpp"

	static std::vector<std::uint8_t> testPool;

	testPool = isobus::IOPFileInterface::read_iop_file("VT3TestPool.iop");

	if (0 != testPool.size())
	{
		std::cout << "Loaded object pool from VT3TestPool.iop" << std::endl;
	}
	else
	{
		std::cout << "Failed to load object pool from VT3TestPool.iop" << std::endl;
	}

	TestVirtualTerminalClient->set_object_pool(0, isobus::VirtualTerminalClient::VTVersion::Version3, testPool.data(), testPool.size());

Note how `testPool` is static here. This is not required, but what is required is that whatever pointer you pass into the VT client via `set_object_pool` MUST remain valid (IE, not deleted or out of scope) during the object pool upload or your application may crash.

If your object pool is too large to store in memory, or you are on an embedded platform with limited resources, you may instead want to use the `register_object_pool_data_chunk_callback` method instead which will get smaller chunks of data from you as the upload proceeds.
This can be used to read from some external device if needed in segments or just to save RAM.

Now, the CAN stack should be able to upload the pool to the VT! But we haven't added any actual application logic yet - we've just set up the communication.
In the next section we'll actually make the on-screen objects functional.

VT Application Layer
---------------------

Now that we've got our VT client configured, let's go over how to use the VT client in your application.

Button and Softkey Events
^^^^^^^^^^^^^^^^^^^^^^^^^^^

One of the main ways you'll get feedback from the VT is that it sends a message whenever a button is pressed, released, held, or aborted (pressed but not released).
You will want to set up processing of these events in your program so that you can take some action based on these events.

To do this, let's create a callback that follows the definition of :code:`VTKeyEventCallback`.

.. code-block:: c++

	// This callback will provide us with event driven notifications of button presses from the stack
	void handleVTButton(isobus::VirtualTerminalClient::KeyActivationCode keyEvent, std::uint8_t, std::uint16_t objectID, std::uint16_t, isobus::VirtualTerminalClient *)
	{
		static std::uint32_t exampleNumberOutput = 214748364;

		switch (keyEvent)
		{
			case isobus::VirtualTerminalClient::KeyActivationCode::ButtonUnlatchedOrReleased:
			{
				switch (objectID)
				{
					case Plus_Button:
					{
						TestVirtualTerminalClient->send_change_numeric_value(ButtonExampleNumber_VarNum, ++exampleNumberOutput);
					}
					break;

					case Minus_Button:
					{
						TestVirtualTerminalClient->send_change_numeric_value(ButtonExampleNumber_VarNum, --exampleNumberOutput);
					}
					break;

					case alarm_SoftKey:
					{
						TestVirtualTerminalClient->send_change_active_mask(example_WorkingSet, example_AlarmMask);
					}
					break;

					case acknowledgeAlarm_SoftKey:
					{
						TestVirtualTerminalClient->send_change_active_mask(example_WorkingSet, mainRunscreen_DataMask);
					}
					break;
				}
			}
			break;

			default:
			{
		
			}
			break;
		}
	}

We'll use this function as our callback. This function will first look at the `key event <https://delgrossoengineering.com/isobus-docs/classisobus_1_1VirtualTerminalClient.html#a5d002c96e4343f2aac9abecb03cc7873>`_ the VT is reporting to decide what to do. In this example, we're doing all actions based on when a button or softkey is `released`.
Next, we switch on the object ID of the button, and from there we can decide what to do about the event.

As an example, if someone presses the softkey :code:`alarm_SoftKey`, this function will be called with the :code:`KeyActivationCode::ButtonPressedOrLatched` event, which we will ignore.
Then, when the user releases the softkey, this function will be called with the :code:`KeyActivationCode::ButtonUnlatchedOrReleased`. If the user hold down the button, the function will be called many times with the :code:`KeyActivationCode::ButtonStillHeld` event.
The :code:`objectID` tells us which button or key was pressed, so putting it all together, when we get the :code:`KeyActivationCode::ButtonUnlatchedOrReleased` event with the object ID of :code:`alarm_SoftKey`, we will take some action.
In this example, the action we're taking is to change the active mask from the main runscreen data mask :code:`mainRunscreen_DataMask` to an alarm mask called :code:`example_AlarmMask`. This will cause a pop-up on the VT (and possibly some beeping if your VT has a speaker).

Note, when we added handling for the :code:`Minus_Button` and :code:`Plus_Button` in the object pool, we are incrementing and decrementing a variable in the program so that we can keep track of the displayed value on the screen.
This variable starts out with a value of 214748364 because the output number in the object pool has an offset applied to it of -214748364. This essentially means that if we send the VT a value of 214748364, it will be shown as 0. If we subtract 1, it will be shown as -1, or if we add 1 it will be shown as 1.
This is how a VT handles negative numbers.

Now that we have our callback, we have to tell the VT client about it so that it knows to call it when appropriate.

.. code-block:: c++
	
	TestVirtualTerminalClient->RegisterVTButtonEventCallback(handleVTButton);
	TestVirtualTerminalClient->RegisterVTSoftKeyEventCallback(handleVTButton);

You could in theory have seperate callbacks for softkeys and buttons, but in this example we're just re-using the same callback for both.

Other Actions
^^^^^^^^^^^^^^

You have the ability to do a lot more than just react to button presses. The VT client exposes many functions that you can call to make the VT do things with your object pool. We've already used a few of them in our callback, such as:

* Changing the active mask
* Changing a numeric value

Check out the `API documentation <https://delgrossoengineering.com/isobus-docs/classisobus_1_1VirtualTerminalClient.html>`_ (or the header file isobus_virtual_terminal_client.hpp) for the full list of functionality available, but the most commonly used ones are listed below along with the name of the function to use on the API.

* Changing the active mask
	* :code:`send_change_active_mask`
* Changing a numeric value
	* :code:`send_change_numeric_value`
* Changing a string value
	* :code:`send_change_string_value`
* Changing a soft key mask
	* :code:`send_change_softkey_mask`
* Changing a list item
	* :code:`send_change_list_item`
* Changing an attribute, such as hiding a container
	* :code:`send_change_attribute`

Final Result
--------------

Here's the final code for this example:

.. code-block:: c++

	#include "isobus/hardware_integration/can_hardware_interface.hpp"
	#include "isobus/hardware_integration/socket_can_interface.hpp"
	#include "isobus/isobus/can_general_parameter_group_numbers.hpp"
	#include "isobus/isobus/can_network_manager.hpp"
	#include "isobus/isobus/can_partnered_control_function.hpp"
	#include "isobus/isobus/isobus_virtual_terminal_client.hpp"
	#include "isobus/utility/iop_file_interface.hpp"
	#include "objectPoolObjects.h"

	#include <csignal>
	#include <iostream>
	#include <memory>

	static std::shared_ptr<isobus::InternalControlFunction> TestInternalECU = nullptr;
	static std::shared_ptr<isobus::PartneredControlFunction> TestPartnerVT = nullptr;
	static std::shared_ptr<isobus::VirtualTerminalClient> TestVirtualTerminalClient = nullptr;
	std::vector<isobus::NAMEFilter> vtNameFilters;
	const isobus::NAMEFilter testFilter(isobus::NAME::NAMEParameters::FunctionCode, static_cast<std::uint8_t>(isobus::NAME::Function::VirtualTerminal));
	static std::vector<std::uint8_t> testPool;
	static SocketCANInterface canDriver("can0");
	static std::uint32_t exampleNumberOutput = 214748364;

	using namespace std;

	void signal_handler(int signum)
	{
		CANHardwareInterface::stop();
		if (nullptr != TestVirtualTerminalClient)
		{
			TestVirtualTerminalClient->terminate();
		}
		exit(signum);
	}

	void update_CAN_network()
	{
		isobus::CANNetworkManager::CANNetwork.update();
	}

	void raw_can_glue(isobus::HardwareInterfaceCANFrame &rawFrame, void *parentPointer)
	{
		isobus::CANNetworkManager::CANNetwork.can_lib_process_rx_message(rawFrame, parentPointer);
	}

	// This callback will provide us with event driven notifications of button presses from the stack
	void handleVTButton(isobus::VirtualTerminalClient::KeyActivationCode keyEvent, std::uint8_t, std::uint16_t objectID, std::uint16_t, isobus::VirtualTerminalClient *)
	{
		switch (keyEvent)
		{
			case isobus::VirtualTerminalClient::KeyActivationCode::ButtonUnlatchedOrReleased:
			{
				switch (objectID)
				{
					case Plus_Button:
					{
						TestVirtualTerminalClient->send_change_numeric_value(ButtonExampleNumber_VarNum, ++exampleNumberOutput);
					}
					break;

					case Minus_Button:
					{
						TestVirtualTerminalClient->send_change_numeric_value(ButtonExampleNumber_VarNum, --exampleNumberOutput);
					}
					break;

					case alarm_SoftKey:
					{
						TestVirtualTerminalClient->send_change_active_mask(example_WorkingSet, example_AlarmMask);
					}
					break;

					case acknowledgeAlarm_SoftKey:
					{
						TestVirtualTerminalClient->send_change_active_mask(example_WorkingSet, mainRunscreen_DataMask);
					}
					break;
				}
			}
			break;

			default:
			{
		
			}
			break;
		}
	}

	void setup()
	{
		CANHardwareInterface::set_number_of_can_channels(1);
		CANHardwareInterface::assign_can_channel_frame_handler(0, &canDriver);

		if ((!CANHardwareInterface::start()) || (!canDriver.get_is_valid()))
		{
			std::cout << "Failed to connect to the socket. The interface might be down." << std::endl;
		}

		CANHardwareInterface::add_can_lib_update_callback(update_CAN_network, nullptr);
		CANHardwareInterface::add_raw_can_message_rx_callback(raw_can_glue, nullptr);

		std::this_thread::sleep_for(std::chrono::milliseconds(250));

		isobus::NAME TestDeviceNAME(0);

		// Make sure you change these for your device!!!!
		// This is an example device that is using a manufacturer code that is currently unused at time of writing
		TestDeviceNAME.set_arbitrary_address_capable(true);
		TestDeviceNAME.set_industry_group(1);
		TestDeviceNAME.set_device_class(0);
		TestDeviceNAME.set_function_code(static_cast<std::uint8_t>(isobus::NAME::Function::SteeringControl));
		TestDeviceNAME.set_identity_number(2);
		TestDeviceNAME.set_ecu_instance(0);
		TestDeviceNAME.set_function_instance(0);
		TestDeviceNAME.set_device_class_instance(0);
		TestDeviceNAME.set_manufacturer_code(64);
		vtNameFilters.push_back(testFilter);

		testPool = isobus::IOPFileInterface::read_iop_file("VT3TestPool.iop");

		if (0 != testPool.size())
		{
			std::cout << "Loaded object pool from VT3TestPool.iop" << std::endl;
		}
		else
		{
			std::cout << "Failed to load object pool from VT3TestPool.iop" << std::endl;
		}

		TestInternalECU = std::make_shared<isobus::InternalControlFunction>(TestDeviceNAME, 0x1C, 0);
		TestPartnerVT = std::make_shared<isobus ::PartneredControlFunction>(0, vtNameFilters);
		TestVirtualTerminalClient = std::make_shared<isobus::VirtualTerminalClient>(TestPartnerVT, TestInternalECU);
		TestVirtualTerminalClient->set_object_pool(0, isobus::VirtualTerminalClient::VTVersion::Version3, testPool.data(), testPool.size());
		TestVirtualTerminalClient->RegisterVTButtonEventCallback(handleVTButton);
		TestVirtualTerminalClient->RegisterVTSoftKeyEventCallback(handleVTButton);
		TestVirtualTerminalClient->initialize(true);
		std::signal(SIGINT, signal_handler);
	}

	int main()
	{
		setup();

		while (true)
		{
			// CAN stack runs in other threads. Do nothing forever.
			std::this_thread::sleep_for(std::chrono::milliseconds(1000));
		}

		CANHardwareInterface::stop();
		return 0;
	}

That's it for this Tutorial. You should be able to run it as long as you have a VT and a "can0" network, see the test pool be uploaded, and be able to interact with all buttons on screen!

If you would like to see more advanced VT tutorials or have other feedback, please visit our `GitHub page <https://github.com/ad3154/ISO11783-CAN-Stack>`_ and feel free to open a discussion! We're friendly, we promise.
