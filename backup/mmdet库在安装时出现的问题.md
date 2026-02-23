```bash
  Installing build dependencies ... done
  Getting requirements to build wheel ... error
  error: subprocess-exited-with-error
  
  × Getting requirements to build wheel did not run successfully.
  │ exit code: 1
  ╰─> [17 lines of output]
      Traceback (most recent call last):
        File "/data/lg/UniAD/uniad2/lib/python3.9/site-packages/pip/_vendor/pyproject_hooks/_in_process/_in_process.py", line 389, in <module>
          main()
        File "/data/lg/UniAD/uniad2/lib/python3.9/site-packages/pip/_vendor/pyproject_hooks/_in_process/_in_process.py", line 373, in main
          json_out["return_val"] = hook(**hook_input["kwargs"])
        File "/data/lg/UniAD/uniad2/lib/python3.9/site-packages/pip/_vendor/pyproject_hooks/_in_process/_in_process.py", line 143, in get_requires_for_build_wheel
          return hook(config_settings)
        File "/tmp/pip-build-env-5xe49pcc/overlay/lib/python3.9/site-packages/setuptools/build_meta.py", line 333, in get_requires_for_build_wheel
          return self._get_build_requires(config_settings, requirements=[])
        File "/tmp/pip-build-env-5xe49pcc/overlay/lib/python3.9/site-packages/setuptools/build_meta.py", line 301, in _get_build_requires
          self.run_setup()
        File "/tmp/pip-build-env-5xe49pcc/overlay/lib/python3.9/site-packages/setuptools/build_meta.py", line 520, in run_setup
          super().run_setup(setup_script=setup_script)
        File "/tmp/pip-build-env-5xe49pcc/overlay/lib/python3.9/site-packages/setuptools/build_meta.py", line 317, in run_setup
          exec(code, locals())
        File "<string>", line 9, in <module>
      ModuleNotFoundError: No module named 'torch'
      [end of output]
  
  note: This error originates from a subprocess, and is likely not a problem with pip.
ERROR: Failed to build 'mmdet3d' when getting requirements to build wheel
```
请使用
``
pip install --no-build-isolation -v mmdet==2.26.0 mmsegmentation==0.29.1 mmdet3d==1.0.0rc6
``
这主要是因为构建环境隔离问题

2、

```bash
 File "/tmp/pip-build-env-9dlzmeme/overlay/lib/python3.9/site-packages/setuptools/build_meta.py", line 317, in run_setup
      exec(code, locals())
    File "<string>", line 6, in <module>
  ModuleNotFoundError: No module named 'pkg_resources'
  error: subprocess-exited-with-error
  
  × Getting requirements to build wheel did not run successfully.
  │ exit code: 1
  ╰─> No available output.
  
  note: This error originates from a subprocess, and is likely not a problem with pip.
```
请使用
``
pip install 'setuptools<70'
``
版本太高导致的