=============================
OpenStack Shared File Systems
=============================

The OpenStack Shared File Systems service (manila) provides shared file systems
to virtual machines, bare metal hosts, and containers.

The shared filesystem service consists of the following components:

``manila-api`` service
  Accepts API requests, and routes them to the manila-share for
  action.

``manila-scheduler`` service
  Responsible for scheduling/routing requests to the appropriate manila-share
  service. It does that by picking one back-end while filtering all except
  one back-end.

``manila-share`` service
  Responsible for managing shared file system devices, specifically
  the back-end devices.
