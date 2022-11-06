================================================
Fine Calibrations with Pulse Simulator
================================================
The amplitude of a pulse can be precisely calibrated using
error amplifying gate sequences. These gate sequences apply 
the same gate a variable number of times. Therefore, if each gate
has a small error dtheta  in the rotation angle then 
a sequence of n gates will have a rotation error of n*dtheta.

Since our new pulse simulator cache the calibrations that it has already solved
in the form of dictionary in which keys are a tuple of instruction name, qubits and parameters
and the vaule is a unitary matrix, it is efficient when simulating a circuit
which consists of same repeated gates. 
Therefore, fine amplitude calibration experiment which is a typical example of
containing such repeated gates within a circuit, can be run efficiently in the "IQPulseBackend"

Let's start with importing some necessary modules and build our pulse simulator with
"SingleTransmonTestBackend" which is inherited from "IQPulseBackend"

.. jupyter-execute:: 

    import numpy as np
    from qiskit.pulse import InstructionScheduleMap
    import qiskit.pulse as pulse
    from qiskit_experiments.library import FineXAmplitude, FineSXAmplitude
    from qiskit_experiments.test.iq_pulse_backend import SingleTransmonTestBackend

.. jupyter-execute::

    backend = SingleTransmonTestBackend()
    qubit = 0
-----------------------------------------------------
Fine X gate Amplitude Calibration
-----------------------------------------------------
We will run the fine calibration experiments with our own pulse schedules. 
To do this we create an instruction to schedule map which we populate with 
the schedules we wish to work with. This instruction schedule map is then 
given to the transpile options of the calibration experiments so that 
the Qiskit transpiler can attach the pulse schedules to the gates in the experiments. 
We will base all our pulses on the default X pulse of "SingleTransmonTestBackend".

.. jupyter-execute::
    x_pulse = backend.defaults().instruction_schedule_map.get('x', (qubit,)).instructions[0][1].pulse
    d0, inst_map = pulse.DriveChannel(qubit), InstructionScheduleMap()

    for name, pulse_ in [("x", x_pulse)]:
        with pulse.build(name=name) as sched:
            pulse.play(pulse_, d0)

        inst_map.add(name, (qubit,), sched)

We now take the ideal x pulse amplitude reported by the backend and 
add/subtract a 2% over/underrotation to it by scaling the ideal amplitude and see 
if the experiment can detect this over/underrotation. We replace the default X pulse 
in the instruction schedule map with this over/underrotated pulse.

.. jupyter-execute::
    ideal_amp = x_pulse.amp
    over_amp = ideal_amp*1.02
    under_amp = ideal_amp*0.98
    print(f"The reported amplitude of the X pulse is {ideal_amp:.4f} which we set as ideal_amp. we use {over_amp:.4f} amplitude for overroation pulse and {under_amp:.4f} for underrotation pulse")

-----------------------------------------------------------------------------------
Detecting an over & under rotated pulse
-----------------------------------------------------------------------------------

.. jupyter-execute::
    # build the over rotated pulse and add it to the instruction schedule map
    with pulse.build(backend=backend, name="x") as x_over:
        pulse.play(pulse.Drag(x_pulse.duration, over_amp, x_pulse.sigma, x_pulse.beta), d0)
    inst_map.add("x", (qubit,), x_over)

    # do the experiment
    overamp_cal = FineXAmplitude(qubit, backend=backend)
    overamp_cal.set_traspile_options(inst_map=inst_map)
    exp_data_over = overamp_cal.run(backend)
    exp_data_over.figure(0)

.. jupyter-execute::
    # build the under rotated pulse and add it to the instruction schedule map
    with pulse.build(backend=backend, name="x") as x_under:
        pulse.play(pulse.Drag(x_pulse.duration, under_amp, x_pulse.sigma, x_pulse.beta), d0)
    inst_map.add("x", (qubit,), x_under)

    # do the experiment
    underamp_cal = FineXAmplitude(qubit, backend=backend)
    underamp_cal.set_traspile_options(inst_map=inst_map)
    exp_data_under = underamp_cal.run(backend)
    exp_data_under.figure(0)

.. jupyter-execute::
    # analyze the results
    target_angle = np.pi
    dtheta_over = exp_data_over.analysis_results("d_theta").value.nominal_value
    scale_over = target_angle / (target_angle + dtheta_over)
    dtheta_under = exp_data_under.analysis_results("d_theta").value.nominal_value
    scale_under = target_angle / (target_angle + dtheta_under)
    print(f"The ideal angle is {target_angle:.2f} rad. We measured a deviation of {dtheta_over:.3f} rad in over-rotated pulse case.")
    print(f"Thus, scale the {over_amp:.4f} pulse amplitude by {scale_over:.3f} to obtain {over_amp*scale_over:.5f}.")
    print(f"On the other hand, we measued a deviation of {dtheta_under:.3f} rad in under-rotated pulse case.")
    print(f"Thus, scale the {under_amp:.4f} pulse amplitude by {scale_under:.3f} to obtain {under_amp*scale_under:.5f}.")