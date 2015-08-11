# CAN FIREWALL

There are two tables, IVI2VEH and VEH2IVI, specifiying rules for
traffic going from the IVI to the vehicle and from vehicle to IVI.


Each one of the two tables has a number of rules.

Each rule has a numberic ID. Rules are executed in order of ascending IDs.

Each rule has an Frame ID mask and a Frame ID filter, applied
according to the standard CAN bus filter principle, as described 
[here](http://www.cse.dmu.ac.uk/~eg/tele/CanbusIDandMask.html).

Below is pseudo code for frame processing.

    process_frame(frame_id, payload) {

      // Traverse all rules
      for (i in rules) {
	    masked_id = (frame_id & rules[i].id_mask);

	    // If the masked out ID is 0, we have a hit. Process
        if (!masked_id) {
	      process(frame_id, input, rules[i].action,
		          rules[i].id_operand, rules[i].data_operand);
	      return;
	    }

        // Check if we have a hit on the masked ID and the
		// ID filter.
	    // If so, we have a hit and will process
	    if (masked_id & rules[i].id_filter) {
	      process(frame_id, input, rules[i].action,
		          rules[i].id_operand, rules[i].data_operand);
	      return;
	    }

        // No hit. Try next rule
	  }

	  // No rule matched. Forward unmodified frame
      process(frame_id, input, ID_AND_DATA_AND,
	          0xFFFFFFFF, 0xFFFFFFFFFFFFFFFF);

	  return ok;
    }




Below is an example of a rule table.

ID | IdMask     |   IdFilter | Action          | IdOperand  | DataOperand        |
---|------------|------------|-----------------|------------|--------------------|
1  | 0xFFFFFFFF | 0x00000012 | ID_AND_DATA_OR  | 0x00FFFFFF | 0x000000FFFFFFFFFF |
2  | 0xFFFFFFF0 | 0x00000120 | ID_SET_DATA_SET | 0x01234567 | 0x0123456780ABCDEF |
99 | 0x00000000 | 0x00000000 | BLOCK           | 0x00000000 | 0x0000000000000000 |


* **RULE 1**<br>
  Rule ID 1 specifies will filter out the lower all bits of the CAN
  frame ID and match those against the specific ID 0x12, thus intercepting
  all CAN frames with the exact ID of 0x00000012.<br>
  An intercepted frame will be forwarded to the other side after ANDing
  the incoming Frame ID with ```0x00FFFFFF```, and then ORing the
  data payload with ```0x000000FFFFFFFFFF```.<br>
  Once the intercepted frame has been forwarded, the next frame is processed.

* **RULE 2**<br>

  Rule ID 2 is evaluated for all frames not intercepted by Rule 1 above.
  The rule will mask the incoming CAN frame ID against ```0xFFFFFFF0```
  and filter the result against ```0x00000120```, intercepting all
  CAN frames between ```0x00000120``` and ```0x0000012F```.<br>
  For each frame intercetpted by rule 2, a fixed frame will be sent to the other side
  with the frame ID ```0x01234567``` and payload ```0x0123456780ABCDEF```.

* **RULE 99**<br>
  Rule 99, a catch all, has a mask that intercepts all frames not intercepted
  by rule 1 or 2. Since the mask is ```0x00000000```, the filter will not be
  used since there no bits will pass the mask to be applied to the filter.<br>
  Rule 99 will block the frame and not send anything to the other side.<br>
  The modification bitmasks for the frame ID and data payload are unused 
  for blocking rules.


In case no rule intercepts the frame, it will be forwarded to the other side unmodified.


CAN FRAME FORMAT 
