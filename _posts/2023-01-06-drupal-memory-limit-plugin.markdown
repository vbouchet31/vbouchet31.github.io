---
layout: post
title:  "Create a plugin for Memory Limit Policy"
date:   2023-01-06 17:00:00 +0100
categories: [drupal]
---
This small post is to describe how to create a plugin for [Memory Limit Policy module](https://www.drupal.org/project/memory_limit_policy).

## The context
For this exercise, we will assume that our site has a content type "Article" which has a "Tags" field. This "Tags" field
is an entity reference to one or multiple taxonomy terms from the "Tags" vocabulary. If the field is filled, a
process is triggered to determine the best articles to suggest based on the selected tags. Because of the complexity of
this process, it appears that if the field has more than 2 values, the default PHP memory_limit of 128M is not enough.

In that context, we can't simply use the "Path" constraint plugin from memory limit policy as we would like to increase
the memory limit based on the "tags" field being filled with more than 2 values.

## The plugin
Assuming we have a module `project_article` to handle the custom code around the article feature. We first need to add
`memory_limit_policy` as a dependency in `project_article.info.yml`.

Then, we create a `src/Plugin/MemoryLimitConstraint/taggedArticles.php` file in `project_article` module. The class
`taggedArticles` must extends `MemoryLimitConstraintBase` and implement the `ContainerFactoryPluginInterface` interface.
It must implement the `create`, `getSummary` and `evaluate` methods. Constraint plugins are [based on annotations](https://www.drupal.org/docs/drupal-apis/plugin-api/annotations-based-plugins).
To register the plugin, we must give an id, a title and a description. As a first step, we will simply make a constraint
which always applies. This is the code which we should have at this step:

```php
<?php

namespace Drupal\project_article\Plugin\MemoryLimitConstraint;

use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\memory_limit_policy\Annotation\MemoryLimitConstraint;
use Drupal\memory_limit_policy\MemoryLimitConstraintBase;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Custom Memory limit constraint for articles.
 *
 * @MemoryLimitConstraint(
 *   id = "tagged_articles",
 *   title = @Translation("Tagged articles"),
 *   description = @Translation("Apply for articles which the 'tags' field is filled.")
 * )
 */
class TaggedArticles extends MemoryLimitConstraintBase implements ContainerFactoryPluginInterface {

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition) {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition
    );
  }

  /**
   * {@inheritdoc}
   */
  public function getSummary() {
    return $this->t('Apply to articles which the "tags" field is filled.');
  }

  /**
   * {@inheritdoc}
   */
  public function evaluate() {
    return TRUE;
  }
}
```

We now need to update the `evaluate` method to return `TRUE` only if the current page is an `article` node and if the `tags` field
has more than 2 values. We can use the [`current_route_match`](https://api.drupal.org/api/drupal/core%21core.services.yml/service/current_route_match/10) service from
Drupal core to determine the route and get the parameters we need to determine if the constraint applies or not.
After updating the `create` method to invoke the `current_route_match` service, defining a `__construct` method to access the service and
updating the `evaluate` method, our `TaggedArticles.php` plugin looks like this:

```php
<?php

namespace Drupal\project_article\Plugin\MemoryLimitConstraint;

use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\Routing\RouteMatchInterface;
use Drupal\memory_limit_policy\Annotation\MemoryLimitConstraint;
use Drupal\memory_limit_policy\MemoryLimitConstraintBase;
use Drupal\node\Entity\Node;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Custom Memory limit constraint for articles.
 *
 * @MemoryLimitConstraint(
 *   id = "tagged_articles",
 *   title = @Translation("Tagged articles"),
 *   description = @Translation("Apply for articles which the 'tags' field is filled.")
 * )
 */
class TaggedArticles extends MemoryLimitConstraintBase implements ContainerFactoryPluginInterface {

  /**
   * Route match service.
   *
   * @var \Drupal\Core\Routing\RouteMatchInterface
   */
  protected $routeMatch;

  /**
   * Constructs constraint plugin.
   *
   * @param array $configuration
   *   A configuration array containing information about the plugin instance.
   * @param string $plugin_id
   *   The plugin_id for the plugin instance.
   * @param mixed $plugin_definition
   *   The plugin implementation definition.
   * @param RouteMatchInterface $route_match
   *   The route match service.
   */
  public function __construct(array $configuration, $plugin_id, $plugin_definition, RouteMatchInterface $route_match) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);

    $this->routeMatch = $route_match;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition) {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('current_route_match')
    );
  }

  /**
   * {@inheritdoc}
   */
  public function getSummary() {
    return $this->t('Apply to articles which the "tags" field is filled.');
  }

  /**
   * {@inheritdoc}
   */
  public function evaluate() {
    $route_name = $this->routeMatch->getRouteName();

    if ($route_name === 'entity.node.canonical') {
      $node = $this->routeMatch->getParameter('node');

      if ($node instanceof Node && $node->bundle() === 'article') {
        $terms = $node->get('field_tags')->getValue();

        // Apply the constraint if "field_tags" has more than 2 values.
        if (count($terms) > 2) {
          return TRUE;
        }
      }
    }
    return FALSE;
  }
}
```

At this point, we can configure a policy as described in [this post](/drupal/2022/12/01/drupal-memory-limit.html) and confirm
it applies properly by using the debug headers.

However, this exercise would not be exhaustive without covering the configuration form. As of now, our plugin does not require
any specific settings. When we select it, the configuration form only shows the "Negate the constraint" option which
is provided by default. Let's push the exercise further and avoid hard coding the limit of 2 "tags" to be referenced for the
constraint to apply.

We need to implement the `buildConfigurationForm` and `submitConfigurationForm` method and define the schema of our configuration.

We create a `config/schema/memory_limit_policy_tagged_articles.schema.yml` file in the `project_article` module. Our
configuration will be very simple as it only contains the number of tags to be filled for the constraint to apply.

```yaml
memory_limit_policy.constraint.plugin.tagged_articles:
  type: memory_limit_policy.constraint.plugin
  mapping:
    tags_threshold:
      type: integer
      label: 'Number of tags threshold'
```

The configuration form only have an input field to enter the number of tags. We can update the `getSummary` method so,
it displays the entered value. We finally need to update the `evaluate` method to use the configured value as the
threshold. Here is the final code of our plugin:

```php
<?php

namespace Drupal\project_article\Plugin\MemoryLimitConstraint;

use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\Core\Routing\RouteMatchInterface;
use Drupal\memory_limit_policy\Annotation\MemoryLimitConstraint;
use Drupal\memory_limit_policy\MemoryLimitConstraintBase;
use Drupal\node\Entity\Node;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * Custom Memory limit constraint for articles.
 *
 * @MemoryLimitConstraint(
 *   id = "tagged_articles",
 *   title = @Translation("Tagged articles"),
 *   description = @Translation("Apply for articles which the 'tags' field is filled.")
 * )
 */
class TaggedArticles extends MemoryLimitConstraintBase implements ContainerFactoryPluginInterface {

  /**
   * Route match service.
   *
   * @var \Drupal\Core\Routing\RouteMatchInterface
   */
  protected $routeMatch;

  /**
   * Constructs constraint plugin.
   *
   * @param array $configuration
   *   A configuration array containing information about the plugin instance.
   * @param string $plugin_id
   *   The plugin_id for the plugin instance.
   * @param mixed $plugin_definition
   *   The plugin implementation definition.
   * @param RouteMatchInterface $route_match
   *   The route match service.
   */
  public function __construct(array $configuration, $plugin_id, $plugin_definition, RouteMatchInterface $route_match) {
    parent::__construct($configuration, $plugin_id, $plugin_definition);

    $this->routeMatch = $route_match;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container, array $configuration, $plugin_id, $plugin_definition) {
    return new static(
      $configuration,
      $plugin_id,
      $plugin_definition,
      $container->get('current_route_match')
    );
  }

  /**
   * {@inheritdoc}
   */
  public function getSummary() {
    return $this->t(
      'Apply to articles which the "tags" field has more than @tags_threshold values.',
      [
        '@tags_threshold' => $this->configuration['tags_threshold'],
      ]
    );
  }

  /**
   * {@inheritdoc}
   */
  public function buildConfigurationForm(array $form, FormStateInterface $form_state) {
    $form = parent::buildConfigurationForm($form, $form_state);

    $form['tags_threshold'] = [
      '#type' => 'number',
      '#title' => $this->t('Tags threshold'),
      '#description' => $this->t('The minimum number of tags to filled in the article for the constraint to apply.'),
      '#default_value' => $this->getConfiguration()['tags_threshold'] ?? 2,
      '#min' => 0,
      '#step' => 1,
    ];
    return $form;
  }

  /**
   * {@inheritdoc}
   */
  public function submitConfigurationForm(array &$form, FormStateInterface $form_state) {
    parent::submitConfigurationForm($form, $form_state);

    $this->configuration['tags_threshold'] = $form_state->getValue('tags_threshold');
  }

  /**
   * {@inheritdoc}
   */
  public function evaluate() {
    $route_name = $this->routeMatch->getRouteName();

    if ($route_name === 'entity.node.canonical') {
      $node = $this->routeMatch->getParameter('node');

      if ($node instanceof Node && $node->bundle() === 'article') {
        $terms = $node->get('field_tags')->getValue();

        // Apply the constraint if "field_tags" has more than the configured values.
        if (count($terms) > $this->configuration['tags_threshold']) {
          return TRUE;
        }
      }
    }
    return FALSE;
  }
}
```